# DAY10 · 系统诊断：sysAK 缺席 → BCC/eBPF 通用底座掉头（2026-09-23）

> 环境：openEuler 24.03 LTS（kernel 6.6.0-145），VM `oe-base`（192.168.252.128），SELinux Enforcing
> 主题：系统诊断工具链。计划指定 sysAK，实际它不在 openEuler 24.03 任何源里——本篇记录"识别计划私货 → 掉头通用 eBPF 底座"全过程，以及 memleak / slabtop / runqlat 三个诊断的完整实战。

## 任务速览

| 计划 | 现实 | 结果 |
|---|---|---|
| `dnf install -y sysak` | 全源 No match（EPOL 开着也没有） | ❌ 24.03 不收录 sysAK |
| 跑通 2-3 个诊断模块看懂输出 | 掉头 BCC/bpftrace | ✅ memleak + slabtop + runqlat |
| —（计划没写的） | 空载 vs 满载调度延迟对照 | ✅ 延迟涨两个数量级，因果闭环 |

## 一、sysAK 为什么装不上（掉头决策）

```bash
sudo dnf install -y sysak          # No match for argument: sysak
grep -nE '^\[|enabled|baseurl' /etc/yum.repos.d/openEuler.repo   # [EPOL] 早存在且 enabled=1
sudo dnf makecache                 # EPOL 元数据拉取成功 = 源活着
dnf list available | grep -i sysak # 空 = 终审：24.03 全源无此包
```

- sysAK（System Analyse Kit）是**龙蜥/阿里社区**工具，不是 openEuler 原生；教程把它写成欧拉自带 = 计划私货
- **EPOL** = Extra Packages for openEuler Linux，类比 RHEL 世界的 EPEL
- 排查三段论：**源开没开（grep repo）→ 活着没活着（makecache）→ 有没有货（list|grep）**。`No match` 只说明"这门没开"或"店里没货"，不说明包不存在
- **dnf search 假阴性**：`dnf search bcc-tools` 报 No matches，但 `list available` 里明明有——search 是文本模糊搜索有分词怪癖，**找包用 `dnf list available | grep`**
- 掉头逻辑：sysAK 底层 = eBPF → **BCC/bpftrace 是全行业通用底座**，学它比学发行版私货更保值

```bash
sudo dnf install -y bcc bcc-tools bpftrace
# 13 包 80MB：llvm-libs-17 / clang / compiler-rt（BCC 现场编译 BPF 程序的底座）
#            + kernel-devel-6.6.0-145（BPF 编译要配套内核头）
rpm -q bcc bcc-tools bpftrace     # 三件 ✅
ls /usr/share/bcc/tools/          # memleak / offcputime / runqlat ...（原始名）
```

**⚠️ openEuler 没有 Fedora 式 `*-bpfcc` 软链** → 一律用全路径 `/usr/share/bcc/tools/<名>`。

## 二、B-1 memleak：进程级内存泄漏检测

```bash
pgrep -a nginx
# 1225 nginx: master process /usr/sbin/nginx
# 1226 / 1227 nginx: worker process
sudo /usr/share/bcc/tools/memleak -p $(pgrep -o nginx) 5 10
```

- **坑**：`pidof nginx` 会把 master+workers 多个 PID 全展开污染 `-p` 参数 → 用 `pgrep -o` 取最老进程（= master）
- **回显**：`Attaching to pid 1225` 后 10 轮 `Top 10 stacks with outstanding allocations:` 全空
- **结论：零未释放分配 = 无泄漏**。nginx 用内存池复用，master 启动后几乎不裸 malloc——**"没泄漏"本身就是诊断结论**
- 系统级佐证：

```bash
sudo slabtop -o | head -20
```

大头是 `avtab_node`（168300 个，SELinux 策略规则表）+ `lsm_inode_cache`（101568 个，inode 安全标签）= **SELinux Enforcing 的内存税**；`ext4_inode_cache`/dentry/inode_cache 是正常文件系统缓存。无异常增长 = 内核 slab 侧没漏，与 memleak 结论互相咬合。

## 三、B-2 runqlat：调度延迟直方图（空载 vs 满载对照）

原理：统计"任务想跑 → 真被调度上 CPU"的等待时间；走 tracepoint **不依赖 PMU，VM 友好**（对照：perf 的 HW 事件在 VMware 常显示 `<not supported>`）。

**空载基线**（`sudo /usr/share/bcc/tools/runqlat 3 8`）：

```
usecs : count   distribution
0->1  : 85      |***************...
8->15 : 146     |***********************...   ← 众数
...
64->127 : 8     |**                            ← 尾部偶发（VM 抢占/时钟抖动）
```

众数 8~15us、长尾 <1ms；两份基线形状一致 = 系统稳定可复现。

**满载对戏（终极单行版）**：

```bash
sudo -v; sudo -n /usr/share/bcc/tools/runqlat 3 8 & sleep 2; \
for i in 1 2 3 4; do dd if=/dev/zero of=/dev/null bs=1M & done; \
sleep 20; pkill -x dd; wait
```

**回显（负载段）**：`1024 -> 2047` 桶爆柱 **2121~2910 count**，`2048 -> 4095` 约 140~160，极端孤点 `32768 -> 65535` 1 次。

**回显（恢复段）**：pkill 杀掉 4 个 dd 后，最后一张表回到空载形状。

| 阶段 | 众数 | 判读 |
|---|---|---|
| 空载 | 8~15 us | 健康 |
| 满载（4×dd） | **1024~2047 us（约 2900 count）** | 涨两个数量级 |
| 恢复 | 0->1 / 8->15 重新主导 | 负载撤→延迟消，因果闭环 |

- **为什么是 1~2ms**：CPU 被占满后，排队 = 等一个**完整 CFS 时间片**（毫秒级）
- **为什么双峰**：0~15us 快桶仍在——运气好碰上瞬间空 CPU 的秒上，排队的等整片
- **30ms+ 孤点**：连续排几个时间片或 VM steal；生产频发就查 CPU 超卖

## 四、坑位速查表

| # | 现象 | 正解 |
|---|---|---|
| 1 | 计划工具装不上（sysAK No match） | 三段排查：源开了吗→源活着吗→源里有货吗；工具缺席就找等价的通用底座 |
| 2 | `dnf search` 说没有 | 假阴性，用 `dnf list available \| grep` |
| 3 | `tee -a` 追加同 repoid 源段 | dnf 静默覆盖不报错；动源前先 grep 全文 |
| 4 | `command -v xxx-bpfcc` not found | openEuler 无软链，用全路径 `/usr/share/bcc/tools/` |
| 5 | `pidof` 多 PID 污染 `-p` | `pgrep -o` 取主进程 |
| 6 | `sudo cmd &` 秒变 Stopped | 后台进程读终端 = SIGTTIN；先 `sudo -v` 喂凭证再 `sudo -n` 禁交互 |
| 7 | 多命令对时被逐行执行打散 | **单行分号链**（`A & sleep 2; B; ...; wait`），人不参与对时 |
| 8 | 密码明文进了截图/终端回显 | 生产 = 凭据泄露；贴图前先扫密码 |

## 五、面试一句话

> "我在 openEuler 上做系统诊断时，计划里的 sysAK 整个发行版都不收录——我没卡死在工具上，而是认准它的底层是 eBPF，用 BCC/bpftrace 的 memleak 和 runqlat 完成了同类诊断：memleak 十轮采样证明 nginx 无泄漏，runqlat 拿到空载 vs 满载的完整对照——CPU 满负荷时调度延迟从十几微秒涨到 1~2 毫秒两个数量级，杀掉负载立刻恢复。工具会缺，方法论不缺。"
