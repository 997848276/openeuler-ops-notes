# day40 · journalctl 与日志排障：日志被谁吃掉了（2026-10-22 · 128 oe-base）

## 一 三把钥匙 + 日志三问

**三把钥匙**：`-u` 按服务｜`-b` 按开机（`-b -1` 上一次）｜`-p` 按级别。
**日志三问**：①谁在写（`_PID`/`-u`）②还剩多少（`--disk-usage`/文件时间）③保留多久（`SystemMaxUse`/`MaxRetentionSec`）。
**查不到 ≠ 没发生**——先验账本，再查事实。

```bash
journalctl --disk-usage                 # → 170.2M
journalctl --list-boots | tail -5       # → 只有 IDX 0（10/21 08:47 起）
sudo journalctl -p err -b --no-pager | tail -20
sudo journalctl -k -b --no-pager | tail -10    # -k = 内核层（带时间戳的 dmesg）
```

## 二 字段检索 + `-u sshd` 的真相

```bash
sudo journalctl -u sshd --since today --no-pager | tail -8
# 实测：全是 sudo 的 PAM 日志，没有一条 Accepted publickey
```

**原因：进程按 cgroup 归 unit**——SSH 登录后跑的 bash/sudo 全在 `sshd.service` 的 cgroup 里，所以 `-u sshd` 把它们都算进来。
⭐ 这是「给 sshd 限资源=锁死自己所有 shell」的镜像证据：**它们真是一家人**。

```bash
sudo journalctl --since "2026-10-22 12:32:20" --until "2026-10-22 12:33:40" --no-pager
sudo journalctl -b _PID=69835 --no-pager          # 字段检索
```
⚠️ 管道里写 `\[` 容易被渲染坏（本次三次翻车），要 pid 就肉眼看。

## 三★★ 日志被谁吃掉了（完整破案链）

| 层 | 证据 | 结论 |
|---|---|---|
| 保留策略 | `/etc/systemd/journald.conf.d/99-persistent.conf`：`SystemMaxUse=200M`、`MaxRetentionSec=4week` | **200M 先撞顶**，4 周保留没机会生效 |
| 实际占用 | 170.2M；`system@*.journal` 每 1h41m 一个 24M，最老只到当天 02:31 | 保留窗口 ≈ **8 小时** |
| 谁在刷 | `grep -c 'nfs-254'` = **116,445**；TOP：systemd 235,977 / sh 27,431 / myapp.sh 21,090 | 三只噪音怪兽 |
| 源 1 | `mnt-nfs\x2d254.automount: Got automount request ... triggered by 1601 (node_exporter)` | **exporter 采 filesystem 指标 → 每次 stat 挂载点触发 automount → 疯狂刷日志** |
| 源 2 | `sh[45426]: 系统巡检报告 ...` | 巡检脚本自己写 journal |
| 源 3 | `myapp.sh` 21,090 条 | 应用脚本刷屏 |

**因果链**：node_exporter 每 15s 抓取 → stat /mnt/nfs-254 → automount 报 already active → 日积 11.6 万条 → 加巡检 2.7 万 → 200M 上限 8 小时撞顶 → 历史被挤掉 → 真出事查不到昨天。

🎯 **最讽刺的自我消耗：监控自己产生的日志，把要查故障的日志挤掉了。**
三条正解：① exporter 排除 autofs（`--collector.filesystem.mount-points-exclude`）；② 脚本输出落文件或 `logger -p info`；③ 显式配 SystemMaxUse + MaxRetentionSec（等保 6 个月）。

## 四★ 日志 TOP 刷屏榜（排障神技）

```bash
sudo journalctl -b --no-pager | awk '{print $5}' | sed 's/\[.*//' | sort | uniq -c | sort -rn | head -8
```
原理：行格式 `Oct 22 02:31:00 host 进程[pid]: ...`，**第 5 列=进程名[pid]**，砍方括号再计数。
👉 **先看谁在刷屏，再查为什么刷**。

## 五 持久化与保留（两个层次）

| 项 | 实测 | 说明 |
|---|---|---|
| `/var/log/journal` | 存在（Oct 5 建） | Storage=auto 下"目录在=持久化开" |
| journald.conf | 只有 `[Journal]` + `ForwardToWall=no` | 容量参数全注释，靠默认 |
| drop-in | `99-persistent.conf`：200M / 4week | **drop-in 优先于主配置** |

**「持久化」≠「历史还在」**：持久化开着，但容量先耗尽照样只留 8 小时。

## 六★ 无害失败识别：mdmonitor

```
systemctl --failed → mdmonitor.service failed (MD array monitor)
cat /proc/mdstat   → Personalities : / unused devices: <none>   ← 没有 RAID
```
- 处置：`systemctl disable --now mdmonitor` + **`systemctl reset-failed mdmonitor`**。
- ⚠️ **disable 只去自启，不清失败状态**（实测 disable 后 `--failed` 仍红，reset 后才 `0 loaded units`）。
- `backup-app.service` 的 `inactive dead` 是**正常**（timer 驱动的一次性任务，跑完就 dead）。

## 七 坑位速查

| # | 现象 | 正解 |
|---|---|---|
| 1 | 查不到昨天日志 | 先 disk-usage + 最老文件时间，保留窗口可能只有几小时 |
| 2 | `-u sshd` 一堆 sudo | 正常：SSH 会话全家住 sshd.service 的 cgroup |
| 3 | `--list-boots` 只有一条 | 未必没持久化，查 drop-in 的 SystemMaxUse 是否被撑爆 |
| 4 | `--failed` 一直红 | 分清真故障/无害失败；disable 后要 reset-failed |
| 5 | 日志刷屏 | 先跑 TOP 榜，再顺藤摸瓜 |
| 6 | 定时任务写 journal | 输出落文件或 logger -p info，别自己挤爆日志 |


