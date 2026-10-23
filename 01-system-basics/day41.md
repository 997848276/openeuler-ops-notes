# day41 · iSula 自愈兜底 + 时钟慢 20 分钟破案（2026-10-23 · 128 oe-base）

> 承接 day41 的日志治理：把"日志被吃"治好后，发现更狠的一层——**容器每天开机就死一次**，
> 以及**这台机的时钟一直是歪的**。今天两条线都收口。

## 一 日志治理收口：exclude 是「替换」不是「追加」

上一次只写了 `--collector.filesystem.fs-types-exclude='^(autofs|...|nfs|nfs4)$'`，
结果 NFS 是排除了，但 **proc/sysfs/tmpfs 全回来了**——因为该参数是**整体替换**默认值。

```bash
# 正解：把默认 23 类整条抄回来，末尾追加 nfs|nfs4
--collector.filesystem.fs-types-exclude='^(autofs|binfmt_misc|bpf|cgroup2?|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|iso9660|mqueue|nsfs|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|selinuxfs|squashfs|sysfs|tracefs|nfs|nfs4)$'
--collector.filesystem.mount-points-exclude='^/(dev|proc|sys|mnt|media|var/lib/docker/.+|host/mnt|host/media)($|/)'
```

### ★ 今天最贵的一张表：四次「我以为治好了」

| # | 当时的"成功" | 真相 |
|---|---|---|
| 1 | 改完 exclude，日志不刷了 | **容器根本没重建**，新参数没生效（老容器还在跑老参数） |
| 2 | metrics 里没有 nfs-254 | **那一刻 NFS 压根没挂载**，没挂载当然没有 |
| 3 | 容器重建了、curl 200 | `-v /:/host:ro` **整行在粘贴时丢失**，exporter 采不到任何指标（"零日志"是因为它没干活） |
| 4 | NFS 挂着 + 参数在场 + 数据里干净 | ✅ 这次才算真治好 |

**验收必须三个前置条件同时成立**：① NFS 处于挂载态 ② 容器实际 Args 里有 exclude ③ metrics 输出里没有该挂载点。

```bash
ls /mnt/nfs-254/ >/dev/null 2>&1; sleep 3; echo "NFS挂载条目数=$(mount | grep -c nfs-254)"
curl -s http://127.0.0.1:9100/metrics | grep '^node_filesystem_size_bytes' | grep -o 'mountpoint="[^"]*"' | sort -u
# → NFS挂载条目数=2 ；mountpoint 只有 / /boot /data /home /run /tmp（6 条，无 nfs-254）
```

## 二★★ iSula 没有 restart policy → systemd 兜底三件套

```bash
sudo last -x reboot | head -4
# reboot  Fri Oct 23 08:42   still running
# reboot  Wed Oct 21 08:47   ...
# ★ 这台机每天开机一次 → 每天所有 iSula 容器死一次
sudo isula ps -a      # zbx-agent2 Exited(255) 9 minutes ago ／ web Exited(255) 4 weeks ago
```

**三件套**：

| 部件 | 内容 |
|---|---|
| 脚本 | `/usr/local/bin/isula-<name>.sh`，幂等：存在 `isula start`，不存在 `isula run` |
| unit | `Type=oneshot` + `RemainAfterExit=yes` + `Requires=isulad.service` + `After=network-online.target` |
| 启用 | `systemctl enable --now isula-<name>.service` |

```ini
[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/bin/isula-node-exporter.sh
ExecStop=/usr/bin/isula stop node-exporter
TimeoutStartSec=180
```

**为什么必须 oneshot**：`isula run -d` 是"下完指令立刻返回"，用默认 `Type=simple` 时 systemd
会盯着那个瞬间退出的父进程判"服务挂了"；`remainAfterExit` 让执行完仍标记 active。

## 三★ 假成功第七、八次：健康检查的「语义」错了

第一版脚本末尾：

```bash
for i in $(seq 1 30); do ss -lnt | grep -q ':10050 ' && exit 0; sleep 1; done
```

unit 显示 `status=0/SUCCESS`，日志里却写着：

```
Error response from daemon: Conflict. The name "zbx-agent2" is already in use by container 6846f5f5...
```

**事件链**：名字判断没命中 → 走 `run` → Conflict → 脚本不退出 → `ss` 看到旧容器还在听 → `exit 0` → systemd 判 Success。

> **健康检查的语义必须是「我的操作成功了吗」，不能是「最终状态看起来 OK 吗」。**
> 🎯 工人装灯泡，发现灯泡已经是好的（装不上），回来勾"完成"——因为他看的是"灯亮"，不是"我装了"。

**处方：每一步显式失败传播**

```bash
isula start "$NAME" || { echo "ERROR: start failed"; exit 1; }
isula run -d ...      || { echo "ERROR: run failed"; exit 1; }
```

改完，同样的 Conflict 立刻变成 `code=exited, status=1/FAILURE`——失败现形了。

## 四★ `--format '{{.Names}}'` 的右填充陷阱

```bash
sudo isula ps -a --format '{{.Names}}' | cat -A
# zbx-agent2  $        ← 带 2 个空格
# node-exporter$       ← 一个都没有
# web         $        ← 带 9 个空格
```

iSula 对**单字段**输出做列宽右填充 → `grep -qx 'zbx-agent2'` 匹配 `zbx-agent2  ` 失败。

- ⚠️ `node-exporter` 之所以通过，**纯粹因为它名字最长（13 字符）没被填充**——同一份逻辑一个成一个败，差别只在名字长度。
- 正解：`NAMES=$(isula ps -a --format '{{.Names}}' | awk '{print $1}')`
- 补充：带 `\t` 的格式（`{{.Names}}\t{{.Status}}`）**不做填充**，所以它没事。

> 通用纪律：**给人看的表格输出拿来做程序判断，必须先按列切分**（`ps`/`ls -l`/`df` 同理）。

## 五★ 三层对质 + 真重启验证

| 层 | 命令 | 判据 |
|---|---|---|
| 文件 | `grep -nE '^(NAME\|NAMES)=\|mount-points-exclude' /usr/local/bin/isula-*.sh` | `NAMES=` 在、exclude 完整 |
| 运行时 | `isula inspect <c> \| python3 -c "...print(Args)"` | 容器实际 Args 与脚本一致 |
| 数据面 | `curl :9100/metrics \| grep node_filesystem_size_bytes` | 无 nfs-254 |

```bash
sudo reboot
# 开机后什么都不做，直接查
uptime; sudo isula ps --format '{{.Names}}\t{{.Status}}'
sudo systemctl is-active isula-node-exporter isula-zbx-agent2
sudo journalctl -u isula-node-exporter -u isula-zbx-agent2 -b --no-pager | tail -8
```

```
09:48:15 Starting iSula container: node-exporter...
09:48:18 isula-node-exporter.sh[1657]: OK: node-exporter ready
09:48:18 Finished iSula container: node-exporter.
09:48:18 isula-zbx-agent2.sh[1659]: OK: zbx-agent2 ready
Up 20 minutes / Up 20 minutes / active / active / *:10050 *:9100
```

**start 复用 vs run 重建**（决定性证据 = 容器 ID）：

```
zbx-agent2     id = 6846f5f535b5b   ← 与重启前 Conflict 报的 6846f5f5 同一个
node-exporter  Created = 08:59:20 < StartedAt = 09:48:14   ← 创建早于本次启动
```

`{{.RunningFor}}` 字段 **iSula 不支持**（`--format not support the field`），字段名靠试不靠猜，拿不到就用 python 解析整份 JSON。

## 六★★ 时钟慢 20 分钟：从算术到根因

### 6.1 起疑：三个数打架

```
uptime:      10:08:36 up 0 min
容器:        Up 20 minutes（StartedAt 09:48:15）
journal -b:  09:48:06 首条 → 10:19:02 末条
```

### 6.2 算术证明「幽灵时间」

```bash
cat /proc/uptime          # 636.92 秒 = 10 分 37 秒
journalctl --list-boots   # boot 0 跨度 30 分 56 秒
```

**30:56 − 10:37 = 20 分 19 秒** ⇒ 不是"过掉的"，是**被拨出来的**。
⭐ `CLOCK_BOOTTIME` **包含挂起时间**，所以 VMware 挂起会让两个数一起变大、差值仍吻合——
**对不上，只剩 `step` 一种可能**，一步排除整轮瞎查。

### 6.3 日志实锤 + 代价

```
09:48:19 chronyd: System clock wrong by 1211.584245 seconds
10:08:31 systemd: Time has been changed
10:08:31 chronyd: System clock was stepped by 1211.584245 seconds
空洞 1212 秒 | 09:48:19 -> 10:08:31      ← journal 时间轴在此断成两截
```

`makestep 1.0 3` 的机制：**开机前 3 次更新里偏差 > 1 秒就直接 step**。
代价是**「时钟准了」和「日志时间轴连续」二选一**——取证时必须配 `--list-boots` + `/proc/uptime` 交叉验证。

### 6.4 `Can't synchronise` ≠ 网络不通

```
^? 139.199.214.202   2   6   375   63   -1174s +/- 108ms
                            ↑
                       reach=375(八进制)=11111101 → 有回包
```

真因：偏差 1174 秒，**已经过了 makestep 允许的窗口，只剩 slewing**，1174 秒要追几个小时 ⇒ 源被判不可用。

```bash
sudo chronyc makestep      # 200 OK，一步到位
date                       # 10:28:55 → 10:48:25（跳 +19分30秒）
chronyc tracking           # Stratum 3 / 139.199.215.251
chronyc sources -v         # ^* 139.199.215.251  -1886us
```

### 6.5 根因在 RTC，不在 chrony

```
rtc_cmos 00:01: setting system clock to 2026-10-23T01:48:06 UTC (1792720086)
                                        ↑ = 09:48:06 CST，而真实是 10:08
```

**内核开机拿 RTC 当系统钟**，而 RTC 本身就慢了 20 分 11 秒。完整闭环：

```
RTC 慢 20 分 11 秒 → 系统钟慢 → chrony makestep 修正 → rtcsync 每 11 分钟把正确时间写回 RTC
```

★ **`rtcsync`**（chrony.conf 里那行）：启用后 chronyd 每 11 分钟回写硬件钟。
本次没做任何 RTC 操作，`hwclock -r` 自己就准了（10:50:26 vs date 10:50:28），就是它在干活。

🔴 **`hwclock -w` 的顺序纪律**：必须**先让系统钟准，再写 RTC**；反了就是把错误固化进下一次开机。

### 6.6 撤回一个判断（打脸 #35）

```
sudo journalctl -b | grep -c "Time has been changed"   # → 2
10:08:31  开机 makestep 1211.58s
10:48:16  手动 chronyc makestep 1157.11s
```

**全程只有两次跳变，两次都对得上。** `10:16:06 Forward time jump detected` 后面并没有对应的
`Time has been changed`，我当时判"第三方夺权"属过度解读。

> **纪律：单条异常日志 ≠ 确认事故。定罪必须有第二条独立证据。**

### 6.7 两个小坑

| 现象 | 真相 |
|---|---|
| `Ref time (UTC)` 比 `date` 慢 60 多秒 | 那是最后采样时刻（Update interval ≈ 65s），不是当前时间 |
| `RMS offset` 刚 step 完几百秒 | 滑动统计的历史残影，当前偏差看 `System time:` |
| cloudflare 源显示 `^-` | 未被选为同步源（偏差 40ms），不是故障 |

## 七 坑位速查

| # | 现象 | 正解 |
|---|---|---|
| 1 | 改了 exclude，指标没变/反而更多 | 该参数是**替换**，把默认 23 类抄回来再追加 |
| 2 | 改完容器参数没生效 | 容器需**重建**；`isula inspect` 验 Args |
| 3 | curl 200 但没指标 | 查 `Binds`——`-v` 那行丢了照样"活着" |
| 4 | 容器开机不自起 | iSula 无 restart policy，写 systemd unit（oneshot + RemainAfterExit） |
| 5 | unit 显示 SUCCESS 实际失败 | 兜底检查看的是端口不是操作结果；每步 `\|\| exit 1` |
| 6 | `grep -qx` 判断容器名不命中 | `--format '{{.Names}}'` 有右填充，先 `awk '{print $1}'` |
| 7 | `--format` 字段报错 | 字段不通用（RunningFor 不支持），改 python 解析 JSON |
| 8 | `uptime` 与容器 `Up` 矛盾 | `Up` 是墙钟差值；真实时长看 `/proc/uptime` |
| 9 | journal 时间轴跳空 | makestep 的 step，查 `Time has been changed` |
| 10 | `Can't synchronise: no selectable sources` | 先看 `reach`，有回包 = 偏差太大没法 step，不是网络问题 |
| 11 | 上千秒偏差同步不上 | `sudo chronyc makestep` 手动 step |
| 12 | 想校准 RTC | 系统钟先准，再 `hwclock -w` |
| 13 | 每次开机钟都歪 | 查 RTC 本身 + `rtcsync` 是否启用 |
| 14 | 看到 `Forward time jump` 就断定有人抢钟 | 必须有第二条证据（`Time has been changed` 计数） |

## 八 面试一句话

> 「这台 openEuler 上我做了两件事。一是给 iSula 容器补自愈：iSula 没有 restart policy，而这台机每天开机一次，等于监控容器天天死——我用 systemd 的 `Type=oneshot` + `RemainAfterExit` 配幂等启动脚本兜底，并且用容器 ID 证明重启后走的是 `start` 复用而不是 `run` 重建。
> 过程中踩了两个坑很典型：一个是脚本的"健康检查"看的是端口在听，不是我的启动操作是否成功，结果 `run` 报名字冲突它照样 exit 0，unit 显示 SUCCESS，实际什么都没干——后来改成每步显式失败传播，失败立刻现形；另一个是 `isula ps --format '{{.Names}}'` 会对短名字做列宽右填充，`grep -qx` 直接不命中，名字长的容器反而侥幸通过，这种"看运气的 bug"最阴。
> 二是查时钟慢 20 分钟：我先用 `/proc/uptime` 和 journal 首末条做算术，因为 `CLOCK_BOOTTIME` 含挂起时间，一下排除了虚拟机挂起，锁定是时钟被 step，再从日志拿到 `stepped by 1211.58 seconds` 实锤；后来 `Can't synchronise` 我也没有直接归因网络，而是看 `reach` 有回包，判断是偏差超过 makestep 窗口只剩 slewing，手动 `chronyc makestep` 恢复；最后顺着内核日志 `setting system clock to ...` 找到根因是 RTC 本身慢了 20 分钟，靠 `rtcsync` 回写硬件钟闭环。这一条最值钱的经验是：**修时钟和保日志时间轴连续是冲突的，而且校准 RTC 必须排在系统钟之后，顺序反了就是把错误固化到下一次开机。**」
