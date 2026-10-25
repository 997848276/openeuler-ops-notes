# day42 · Shell 工程化四课 + SELinux label 实锤（2026-10-24 · 128 oe-base）

> 上午线：开蓝宝书 P0 缺口「Shell 工程化」。四个脚本从"能跑"改造成"能被机器调用"，
> 顺带把昨天欠的 RTC / iSula 自愈两个终验收掉，
> 并在挂 timer **之前**抓到两个真隐患（SELinux label、文件属主）。

## 一 块 0：一个用错的判据（打脸 #39）

昨天约好"第一步 step 距 `uptime -s` 超过 2 分钟就算通过"。今天发现**这个判据本身是坏的**：

```bash
uptime -s        # 2026-10-24 03:53:18  ← 看起来像今天凌晨开过机
```

真实开机是 **10-23 09:48**。原因：

```
uptime -s = 当前墙钟 − 运行时长
挂起期间运行时长不增加，恢复后 wall 又跳走 → 这个值被"推着往前走"
```

> ★ **打脸 #39**：`uptime -s` 在**挂起过的机器上不可用**。
> 想看"今天开没开机"，看 `journalctl -b | grep "Time has been changed"` 的时间分布，或者干脆看容器的 `Up` 时长。

真结论：`node-exporter Up 23 hours`、`zbx-agent2 Up 17 hours` → **今天一次没重启过**，两个终验实际都还悬着。

13 条 `Time has been changed` 全部可解释：夜间合盖 11 小时恢复、实验拨钟、手动 makestep——**没有一条是开机**。

## 二 第 1 课：公共函数库 `lib-common.sh`

```bash
sudo tee /usr/local/bin/lib-common.sh >/dev/null <<'EOF'
#!/bin/bash
set -euo pipefail

log() { printf '%s [%-5s] %s\n' "$(date '+%F %T')" "${1:-INFO}" "${*:2}"; }
die() { log ERROR "$*"; exit 1; }

wait_port() {
    local port=$1 timeout=${2:-30} i=0
    while [ "$i" -lt "$timeout" ]; do
        if ss -lntH "sport = :${port}" | grep -q .; then
            log INFO "port ${port} ready"; return 0
        fi
        sleep 1; i=$((i + 1))
    done
    log ERROR "port ${port} not ready in ${timeout}s"; return 1
}
EOF
```

### `set -euo pipefail` 各管什么

| 选项 | 管什么 | 不管会怎样 |
|---|---|---|
| `-e` | 任一命令非 0 退出就退出脚本 | 中间失败继续跑，最后 `exit 0` —— **假成功** |
| `-u` | 引用未定义变量报错 | `$NAEM` 拼错 → 当空串 → 删错东西 |
| `-o pipefail` | 管道整体取最后一个非 0 | `cmd \| head` 里 `cmd` 崩了，`$?` 还是 0 |

### `source` 会把 shell 选项传给调用方

`set -euo pipefail` 是 **shell 进程级**选项，`source` 在同一进程执行 → **库里写一次，所有脚本继承**。

> 🎯 类比：`source` 不是"找个人替你干活"，是"把条文写进你自己身体里"——它身上的规矩也跟着变你的。

### `ss -lntH "sport = :9100"` 优于 `ss -lnt | grep ':9100 '`

- `-H` 去掉表头，`grep -q .` 才能判"有没有内容"
- 原生过滤不走文本匹配 → **不踩尾空格陷阱**（§四十五）
- 不带空格的 `grep ':9100'` 又会误中 `:91000`

### `set -e` 下禁写 `((i++))`

`i=0` 时 `((i++))` **返回值是 0（假）**，被 `-e` 判失败 → 脚本莫名退出。正解 `i=$((i + 1))`（赋值，不是算术求值命令）。

### 验收

```
语法 OK
[INFO ] port 9100 ready      → 正例 rc=0
[ERROR] port 9999 not ready in 2s  → 反例 rc=1
```

> ★ **函数返回码 ≠ 脚本退出码**：`wait_port` 返回 1，但外层 `bash -c` 没开 `-e`，脚本 `$?` 仍是 `echo` 的 0。

## 三 第 2 课：接回 `isula-node-exporter.sh`（消 6 处毛病）

| # | 原毛病 | 现在 | 为什么是病 |
|---|---|---|---|
| 1 | 无 `set -e/-o pipefail` | 库首声明，source 继承 | 假成功 |
| 2 | 无统一失败入口 | `\|\| die "..."` | 每处手写 `{ echo; exit 1; }`，漏写即假成功 |
| 3 | `NAME` 写死多处 | 顶部变量 | 改名漏一处 = 起两个容器 |
| 4 | `seq 1 30` 等待逻辑抄一遍 | `wait_port` | 第二个脚本还得再抄 6 行 |
| 5 | `echo "ERROR"` 无时间无级别 | `log`/`die` | 分不清是哪次运行 |
| 6 | `exec >>LOG 2>&1` 吞日志 | **删除**，交给 journal | 输出全进文件，journal 一片空白 |

> ⚠️ `exec >>"$LOG" 2>&1` 不是"日志丢了"，是"日志去了你不知道的地方"（10/12 备份断 43h 的破案障碍同款）。

### 四条验收的真实判读

```
① [INFO ] node-exporter already running, skip → port 9100 ready → guard ok   rc=0

② zbx-agent2     bf3be1cde589  Up 17 hours
   node-exporter  a38c5ba7b162  Up 23 hours

③ isula stop 后：node-exporter  Exited (143) 4 seconds ago
   restart 后：  node-exporter  a38c5ba7b162  Up 5 seconds     ← ID 未变

④ journal 出现 isula-node-exporter.sh[11777] 前缀的 4 行 + Finished
```

1. **`Exited (143)` = 128 + 15 = SIGTERM**。密码本：`143=128+15(TERM)`、`137=128+9(KILL)`、`130=128+2(INT)`。143 说明是"被好好请走的"。
2. **ID 未变 = 走了 `isula start` 复用分支**。如果 ID 变了说明走了 `run` **重建**分支——两种结果 `status` 都是 `Up`，**只有 ID 能区分**。→「结果对不代表路径对」。
3. **`xxx.sh[11777]` 前缀是 systemd 自动加的** → 按 cgroup 归属（§四十七同款）。

## 四 第 3 课：抽 `ensure_container()`——先侦察再定签名

原 `isula-zbx-agent2.sh`（19 行）与 node-exporter 的差异只有 4 处：**名字 / 镜像 / 端口 / 三个 `-e`**。

### 侦察揪出两个真 bug

**bug 1：`set -u` 不是 `set -e`（半个防护）**

```bash
set -u; false; echo "还在跑"     # 照样执行
```

`-u` 只防变量名拼错，**完全不防命令失败**。原脚本靠手写 4 处 `|| { echo; exit 1; }` 兜底，漏一处即假成功。

> 🎯 `set -u` 是安全带，`set -e` 是刹车。只系安全带不装刹车，撞车时人被固定住了，车还在往前冲。

**bug 2：二分法漏了「存在且在跑」这一态**

```bash
NAMES=$(isula ps -a --format '{{.Names}}' | awk '{print $1}')   # -a 含已停止
if echo "$NAMES" | grep -qx "$NAME"; then
    isula start "$NAME"      # ← 容器正在跑时也会命中这里
```

对 running 容器 `isula start` 报错 → `|| exit 1` → **systemd 标 failed**。
平时不炸是因为 `systemctl restart` 先跑 ExecStop 把容器停了；**手动执行必炸**。

→ **「存在」≠「需要启动」**。三分支：跑着跳过 / 停了 start / 没了 run。

### 红线：三个 `-e` 一个都不能丢

```
-e ZBX_HOSTNAME=oe-base
-e ZBX_SERVER_HOST=192.168.252.254
-e TZ=Asia/Shanghai        ← 丢了它，agent 回 UTC，图表时间错 8 小时，且不报错
```

> ⚠️ **重构最危险的不是改错逻辑，是悄悄丢参数。**

### 签名：`--` 是歧义消解符

```bash
# ensure_container <name> <image> <port> [timeout] [-- <isula run 额外参数...>]
ensure_container() {
    local name=$1 image=$2 port=$3
    shift 3
    local timeout=30
    if [ $# -gt 0 ] && [ "$1" != "--" ]; then timeout=$1; shift; fi
    if [ $# -gt 0 ] && [ "$1" = "--" ];  then shift; fi
    local extra=("$@")

    if isula ps --format '{{.Names}}' | awk '{print $1}' | grep -qx "$name"; then
        log INFO "$name already running, skip"
    elif isula ps -a --format '{{.Names}}' | awk '{print $1}' | grep -qx "$name"; then
        log INFO "$name stopped, starting"
        isula start "$name" || return 1
    else
        log INFO "$name absent, creating"
        isula run -d --name "$name" --net=host "${extra[@]}" "$image" || return 1
    fi
    wait_port "$port" "$timeout" || return 1
    log INFO "$name ready (port $port)"
}
```

没有 `--`，`ensure_container x img 9100 -e A=1` 里的 `-e` 会被当成 timeout。

### 重构验收：必须逼它走 `run` 分支

```
start 分支：bf3be1cde589 Up 18 hours → bf3be1cde589 Up 18 seconds   ← ID 不变
run   分支：isula rm -f → 新 ID 149435f819ac
           isula exec zbx-agent2 date  → Sat Oct 24 09:53:13 CST 2026   ← TZ 透传成功
           env | grep -E 'ZBX_|TZ'     → 三个 -e 全在 + ZBX_VERSION=7.0.32
```

> ⚠️ **`skip` / `start` 跑一万次也碰不到 `run` 分支**。不做 `rm` 重建，参数拼接就是没验证过的代码。
> 🎯 换了厨房水管，光开水龙头看出水只验了"旧管还通"；得关总阀再开，才知道新接那段漏不漏。

## 五 第 4 课：`isula-guard.sh`——让脚本能被「机器」调用

| 模式 | 谁调用 | 允许改状态 | 退出码 |
|---|---|---|---|
| `--check` | 监控 / timer 判断 / 人工巡检 | **绝对不许** | 0=健康 1=异常 2=用法错 |
| `--fix` | 人工 / timer 自愈 | 可以 | 0=已修复 1=修复失败 |
| `--status` | 人工看一眼 | 不许 | 恒 0 |

> ⚠️ **健康检查带副作用 = 现场被自己破坏**。监控每分钟"检查并修复"→ 容器刚挂 5 秒就修好 → **你永远收不到告警**。
> 🎯 医生查房顺手给每个病人打一针——你永远不知道谁本来有病。

### 四个工程点

1. **`getopts` 只认短选项** → 先把 `--check` 降级成 `-c` 再交给它。开头那个 `:` 让错误走 `\?` 分支由你处理。
2. **关联数组** `declare -A PORTS=( [node-exporter]=9100 ... )`，`set -u` 下取不存在的 key 会报错。
3. **两条判据 AND**：`[ "$st" = running ] && [ "$pt" = up ]`。只看端口 → 端口被别的进程占着会误判；只看容器 → `Up` 但进程卡死端口已不听。
4. **`flock -n`** 防 timer 与人工撞车。锁跟进程走，进程死自动释放——比"建 .lock 文件、退出时删"靠谱（后者被 `kill -9` 留死锁）。

### 五步验收的关键证据

```
① 无参数 → 用法 + rc=2          （2=你调错了，1=东西坏了，不能混）
② CHECK OK rc=0
③ 记指纹：node-exporter 4421507795da / zbx-agent2 149435f819ac
④ stop zbx-agent2 → container=stopped port=10050/down → CHECK FAILED rc=1
⑤ --fix 只动坏的（好的 skip）→ 2 秒修复 → rc=0
终态：zbx-agent2 149435f819ac Up 11 seconds   ← ID 不变
      node-exporter 4421507795da Up 17 minutes ← 全程没被碰
```

> ★ **「无副作用」的验收方法：先记指纹，做一次，再比指纹**——不是靠"我觉得它没动"。

## 六 降噪改造：信噪比

`--fix` 全健康也输出 3 行 → 挂 timer 后 `3×12×24 ≈ 864 行/天`（§四十七/四十八 日志撑爆同款）。

改法：把「健康」抽成**唯一定义处**，`--fix` 先静默体检 → 健康即退 → 只对坏的下手。

```bash
is_healthy() {
    local n=$1
    [ "$(container_state "$n")" = running ] && [ "$(port_state "${PORTS[$n]}")" = up ]
}
```

| 场景 | 改造前 | 改造后 |
|---|---|---|
| 全健康 | 3 行 | **1 行** |
| 1 坏 1 好 | 6 行 | **4 行**（且没有健康容器的 skip 行） |

> ★ 反面教材：`do_check` 写一遍判据、`do_fix` 再写一遍 → **两套"健康"定义**，某天 check 说坏了、fix 说好的，互相打架。
> 🎯 家里三个钟，坏了两个你都不知道信哪个。**"健康"只能有一个定义，其他地方都是引用。**

## 七 RTC 终验：`grep -c = 0` 有两种解释

真重启（09:53，1m22s）后：

```
date; sudo hwclock -r      → 09:55:07 / 09:55:07.997642+08:00    差 < 1 秒
journalctl -b | grep -c "Time has been changed" → 0              （昨天 1 次跳 1211s）
```

### ⚠️ `0 次跳钟` ≠ 「钟是准的」

| 解释 | 含义 | 结论 |
|---|---|---|
| ① RTC 已准，不需要 step | 修复成功 | ✅ |
| ② **chronyd 根本没起来**，没人跳钟 | 假阴性 | ❌ |

**这两件事在日志上长得一模一样。** 补正向证据：

```
systemctl is-active chronyd → active
chronyc tracking  → System time : 0.000469538 seconds slow   （0.47 毫秒，昨天 15590 秒）
chronyc sources   → ^* 119.28.206.193                         （有选中源）
```

> ★ **三层对质**：进程在 + 偏差微秒级 + 有选中源。三条同时成立才能结案。
> 🎯 病人不发烧了，可能是病好了，也可能体温计没电了。**"没症状"≠"健康"。**

### 顺带：`Reach` 是八进制位图，不是次数

| Reach | 十进制 | 二进制 | 最近 8 次成功 |
|---|---|---|---|
| `157` | 111 | `1101111` | 7 次 |
| `77` | 63 | `0111111` | 6 次 |
| `75` | 61 | `0111101` | 5 次 |

⚠️ `Reach=77` 不是"联系 77 次"。**只有 `Reach=0` 才是完全联系不上。**

## 八 iSula 自愈终验：ID 全部不变

```
重启前：zbx-agent2 149435f819ac   node-exporter a38c5ba7b162
重启后：zbx-agent2 149435f819ac   node-exporter a38c5ba7b162
```

**主机真重启了，容器 ID 一个没变** → 流程是：

```
isulad 起来 → 容器 Exited（数据还在）→ systemd oneshot 跑 guard
→ 命中「存在且已停止」→ isula start → 同一个容器被唤醒
```

ID 变了就说明走了 `run` 重建 → **重启把容器弄丢了**。**ID 不变证明的是"容器被正确保存和唤醒"，不只是"容器起来了"。**

## 九 `RemainAfterExit` 的假 active（三预测全中）

```bash
sudo isula rm -f node-exporter                   # 容器没了
sudo systemctl is-active isula-node-exporter     # → active   ← 撒谎
sudo isula ps -a --format '{{.Names}}'           # → 只剩 zbx-agent2
sudo systemctl restart isula-node-exporter       # → 重建，ID 变成 4421507795da
```

```
管理面（systemd）= active     ← "我上次跑成功了"
运行面（isula）  = 空         ← 东西已经被删了
```

> ★ `Type=oneshot` + `RemainAfterExit=yes` **记的是历史，不是现状**。
> 解药不是改 unit，是**多层补偿**：端口探活告警（1~2 分钟兜住）+ 可选 `isula-guard.timer`。
> **运维不是靠某个组件全能，是靠多层补偿。**

## 十★ SELinux label 实锤：`mv` 不换标签

```bash
sudo mv /home/devops/isula-guard.sh /usr/local/bin/isula-guard.sh
```

`/home/devops` 与 `/usr/local/bin` **同一文件系统** → `mv` = `rename(2)` → **inode 不变** →
SELinux label（存在 inode 的 xattr 上）**跟着旧路径走**。

### 四个文件里，只有 `mv` 进来的那个是错的

```
unconfined_u:object_r:user_home_t:s0  isula-guard.sh        ← 只有它错
unconfined_u:object_r:bin_t:s0        isula-node-exporter.sh  （tee 新建，继承目录）
unconfined_u:object_r:bin_t:s0        isula-zbx-agent2.sh
unconfined_u:object_r:bin_t:s0        lib-common.sh
```

| 写法 | label 结果 |
|---|---|
| `mv` 同文件系统 | **保留原 label** |
| `cp` 到目标 | 新 inode → **继承目标目录 default label** |
| `tee` / `>` 新建 | 新 inode → **继承目标目录 default label** |
| 事后修 | `restorecon -v`，不行再 `chcon` |

### `restorecon` vs `chcon`

| 命令 | 依据 | 持久性 |
|---|---|---|
| `restorecon` | **策略里的路径规则** | **持久**，下次结果一样 |
| `chcon` | 你手敲的值，直接写 xattr | **临时**，下次 restorecon 被冲掉 |

> ★ **`chcon` 改的是"这一刻"，`restorecon` 改的是"规矩"。**

### 可复位实验：把 label 改坏，看 systemd 怎么死

```bash
sudo chcon -t user_home_t /usr/local/bin/isula-guard.sh
sudo systemd-run --unit=guard-badlabel --wait --collect --pipe \
    /usr/local/bin/isula-guard.sh -c
```

```
Finished with result: exit-code
Main processes terminated with: code=exited/status=203
Service runtime: 1ms                    ← 正常是 90~113ms

avc: denied { execute } for pid=6392 comm="(guard.sh)" name="isula-guard.sh"
  scontext=system_u:system_r:init_t:s0
  tcontext=unconfined_u:object_r:user_home_t:s0
  tclass=file permissive=0
```

### AVC 定罪只看两个字段

```
scontext ≠ 期望配对   +   permissive=0
    ↓                        ↓
域和标签对不上          策略真的生效了
```

⚠️ `permissive=1` 时日志照样有 AVC，但**根本没拦住**——你会看到"日志说被拒、实际跑通了"。
**AVC 存在 ≠ AVC 生效**（同族：单条异常日志 ≠ 事故）。

### 意外收获：被拒的是 `access()`，不是 `execve`

```
syscall=21  success=no  exit=-13
```

| 号 | 调用 |
|---|---|
| **21** | **`access`**（`a1=2` 就是 X_OK） |
| 59 | `execve` |

systemd-executor 先 `access(path, X_OK)` 探路就被拒 → **压根没走到 `execve`** → 所以 `runtime=1ms`。
报错文案是 **`Permission denied`（EACCES，-13）**，不是 `Operation not permitted`（EPERM）。

> ⚠️ **看到 `Permission denied` 别只想 `chmod`** —— SELinux 拒绝长得一模一样，必须靠 `ausearch` 分开。

### `203` 和 `127` 别混

| 码 | 谁说的 | 含义 |
|---|---|---|
| **203** | systemd（`EXIT_EXEC`） | **"我连程序都启动不了"** |
| **127** | 程序自己（shell） | 启动成功了，但**内部**找不到命令 |

🎯 **`203` 在门外，`127` 在门内。**

## 十一 第二个发现：属主是 `devops:devops`（提权路径）

```
-rwxr-xr-x. 1 devops devops ... isula-guard.sh        ← 属主 devops
-rwxr-xr-x. 1 root   root   ... isula-node-exporter.sh
```

`mv` 把**属主也一起搬过来**（同一 inode）。这个脚本由 root 执行：

```
devops 不需要 sudo 就能改写它 → 下次 root 执行 = 执行 devops 写的代码
```

本机 `devops` 本来有 `NOPASSWD` sudo，风险没增加——但属于「**恰好没炸**」，不是「**安全**」。

```bash
sudo chown root:root /usr/local/bin/isula-guard.sh
sudo chmod 755 /usr/local/bin/isula-guard.sh
# -rwxr-xr-x. 1 root root unconfined_u:object_r:bin_t:s0 3708 Oct 24 10:18
```

> ★ **mtime 仍是 10:18 未变**：`chown`/`chmod`/`chcon` 只改 ctime 和元数据，**不动 mtime**。
> 所以"改完权限时间戳没变"不是没生效，而是**证明只动了元数据、没动内容**。看 ctime 用 `ls -lc` / `stat`。
>
> ★ 铁律：**被 root 执行的脚本，属主必须是 root。**

### 更稳的写法：`install` 代替 `mv`

```bash
sudo install -m 755 -Z /home/devops/isula-guard.sh /usr/local/bin/isula-guard.sh
```

`install` 是**新建文件**（内部 cp + chmod）→ 继承目标目录的 default label，属主为 root。
**一条命令 = 复制 + 权限 + 标签 + 属主。**

## 十二 可复位实验四条纪律

1. **先记录基线**：改之前 `ls -Z` 存一份
2. **改坏只改一处**：保证因果唯一
3. **必须有复位步骤**：`restorecon -v` + `ls -Z` 确认
4. **复位后要终验**：再跑一次 `systemd-run`，`status=0` 才算真复位

> ★ 第 4 条最容易漏：**"我改回去了" ≠ "改回去了"**。跟 `git status` 空才算干净同理。

复位终验：

```
Finished with result: success
Main processes terminated with: code=exited/status=0
Service runtime: 90ms            ← 对比坏标签时的 1ms
rc=0
```

## 十三 坑位速查

| 现象 | 正解 |
|---|---|
| `uptime -s` 显示今天开过机 | 挂起过的机器上它是推算值，不可信 |
| 脚本最后 `exit 0` 但实际失败 | 缺 `set -e` / `\|\| die` |
| `set -u` 写了还是假成功 | `-u` 只防变量名，失败检测要 `-e` |
| `rc=$?` 取到管道末位命令的码 | `${PIPESTATUS[0]}` 或 `set -o pipefail` |
| `((i++))` 让脚本莫名退出 | 改 `i=$((i + 1))` |
| `grep ':9100 '` 漏判 | `ss -lntH "sport = :9100"` 原生过滤 |
| `journalctl -u` 空白 | 脚本里 `exec >>LOG 2>&1` 吞了输出 |
| 停了容器后 `isula ps` 回显空 | 加 `-a`；**空 ≠ 不存在** |
| 容器在跑时手跑脚本就 failed | 二分法漏「存在且在跑」，要三分支 |
| ID 变了但 status 一样 | `start`→ID 不变；`run`→ID 变；**只有 ID 能分辨** |
| 抽函数后时区变 UTC | `-e TZ=` 在重构中丢了，且不报错 |
| 容器删了 `systemctl` 还 active | `RemainAfterExit=yes` 记的是历史 |
| `grep -c "Time has been changed"`=0 就判钟准了 | 0 有两种解释，要补 `chronyc tracking` |
| `Reach=77` 以为联系 77 次 | 八进制位图，`0o77`=6/8 |
| `/usr/local/bin` 下脚本 label 是 `user_home_t` | `mv` 带过来的，`restorecon -v` 修 |
| `sudo` 能跑但 systemd 报 `203` | 真实域 `init_t` 执行被拒，不是权限位问题 |
| 看到 `Permission denied` 只查 `chmod` | SELinux 拒绝同款文案，`ausearch` 才分得清 |
| 脚本属主是普通用户 | root 执行的脚本必须 `root:root` |
