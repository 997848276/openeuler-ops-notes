# day34 · 进程与信号深水区：从 STAT 判读到僵尸收尸（2026-10-13 · 128 oe-base）

## 一 侦察：STAT 状态语言 + /proc 开盖

```bash
ps -eo pid,ppid,stat,comm --sort=pid | head -20
# systemd(1) Ss ｜ kworker 一排 I< ｜ rcu/migration/cpuhp/ksoftirqd
ps -o pid,ppid,stat,wchan:20,cmd -p $$      # bash：Ss，wchan=do_wait
cat /proc/$$/status | head -8               # Umask 0022 / TracerPid 0 / Tgid=Pid
ps -eo stat,pid,comm | grep -E '^(D|Z)' || echo "当前无 D/Z"
```

| 标记 | 含义 | 实例 |
|---|---|---|
| `Ss` | 可中断睡眠 + 会话首进程 | systemd(1)、bash |
| `I<` / `I` | **Idle，内核线程专属** + 高优先级 | kworker / rcu / ksoftirqd |
| `S` | 可中断睡眠 | 大多数在等的进程 |
| `Z` | 僵尸（本章主角） | 自造的 11809 |
| `D` | 不可中断（通常等 IO） | kill -9 杀不掉的第二种答案 |

- ★**看 PPID 认出身**：用户进程祖先=systemd(1)；**PPID=2（kthreadd）=内核线程**。
- ★**内核线程对信号免疫**：`I` 状态的 kworker 你 kill -9 也杀不死——信号机制根本不投递给内核线程。
- ★bash 睡在 `wchan=do_wait`：**shell 天生会收尸**，所以正常命令跑完不留僵尸；造僵尸必须让父进程睡着不收。
- /proc/status 取证两字段：**Umask（实际生效值）**；**TracerPid 非 0 = 被 strace/gdb 挟持**。

## 二 僵尸案：造一个、杀三次、收一次

```bash
# 造（父睡 5 分钟不收尸，子秒死）——单行写法，避免 -c 多行缩进被终端吞
python3 -c "import os,time; pid=os.fork(); print('parent:',os.getpid(),'child:',pid) if pid else os._exit(0); time.sleep(300)" &
ps -eo pid,ppid,stat,comm | grep -E 'python|defunct'   # 11809 11804 Z python3

kill -9 11809        # ① 朝尸体开枪：rc=0，但 Z 依旧——没有东西可杀
kill 11804           # ② 正解：杀父 → [1]+ Terminated → 11809 被 init 收养收尸
ps -o pid,ppid,stat,comm -p 11809                      # ⭐ 查无此人
```

**僵尸案四条判词**：
1. 僵尸 = 死了但没被 wait 的进程：**占 PID 户口，不占内存/CPU**；
2. `kill -9` 僵尸无效——僵尸已经死了，朝尸体开枪白费子弹；
3. 正解 = **父进程 wait 收尸，或父死后由 init(PID 1) 收养**（init 天生收尸）；
4. 预防优于治疗：父进程设好 SIGCHLD/wait，别让尸体堆积耗尽 PID。

**意外收获（手滑送来的案例）**：`kill 1224` → `Operation not permitted`——1224 真身是 **atuned（A-Tune 引擎）**。
- ★**信号权限模型**：普通用户只能给自有进程发信号，root 进程 kill 不动——**权限不只是门槛，本身就是保护**（这一拒拦下了误杀系统服务）。

## 三 TERM vs KILL 对决：犟种的最后 300 秒

```bash
cat > /tmp/stubborn.py <<'EOF'
import signal, time, os
def on_term(signum, frame):
    print("收到 TERM(15)，喊冤但我不走！", flush=True)
signal.signal(signal.SIGTERM, on_term)
print("stubborn pid:", os.getpid(), flush=True)
time.sleep(300)
EOF
python3 /tmp/stubborn.py > /tmp/stubborn.log 2>&1 &
SPID=$(pgrep -f stubborn.py | head -1)

kill $SPID          # TERM：ps 显示 12046 S 还在；log 多一行"喊冤"
kill -9 $SPID       # KILL：[1]+ Killed；ps 查无此人；log 喊冤永远只有一条
```

| | TERM(15) | KILL(9) |
|---|---|---|
| 谁执行 | **进程自己**（收到信号自行处理） | **内核直接回收** |
| 可捕获 | ✅ 可自定义（犟种喊冤不走） | ❌ 完全不可捕获 |
| 能否清场 | ✅ 落盘/关文件/通知下游 | ❌ 不能——**未落盘的内存状态全消失** |
| 适用 | 默认体面离场 | 犟种/死锁/无响应 |

- ★**推理到生产**：`systemctl stop` = 先 TERM → 等 `TimeoutStopSec` → 超时才 KILL。**给体面的服务留收拾行李的时间**。这也是“状态必须落盘”（§三十七）的信号层解释。

## 四 坑位速查

| 坑 | 正解 |
|---|---|
| 多行 `python3 -c` 报 IndentationError | 终端续行吞行首缩进；**多行写文件（heredoc），单行才 -c** |
| `ps -eo stat,comm \| grep defunct` 永远落空 | comm 列只显二进制名；`<defunct>` 在 args 列；**判僵尸看 STAT=Z** |
| `ps \| awk ... \|\| echo` 兜底不触发 | awk 无匹配也 rc=0；兜底用 grep/ps（无匹配 rc=1） |
| kill -9 僵尸无效 | 正常；杀父进程或等 init 收养 |
| kill 系统进程报 Operation not permitted | 信号权限模型；用 sudo 或 systemctl 正规离场 |
| kill -9 内核线程杀不死 | 内核线程对信号免疫 |
| 重定向到文件后 print 看不见 | python 输出缓冲；`print(..., flush=True)` |

## 面试一句话

> 「kill -9 之后进程还在，有三种答案：你杀的是**内核线程**（信号机制不投递给它们，PPID=2 可认出身）、它卡在 **D 状态**等 IO（9 号也排不上队）、或者它是**僵尸**——已经死了，只差父进程一纸 wait 收尸，杀尸体当然无效，正解是让父进程收尸或者等 init 收养。
> 信号本身是权限化的，普通用户只能给自己的进程发信号；TERM 是可捕获的请求、KILL 是内核强制执行且不可拦截——所以 systemd stop 先 TERM 后 KILL，给服务留收拾行李的时间，而**任何依赖进程内存里的状态，在 9 号进场那一刻都会消失，必须提前落盘**。」
