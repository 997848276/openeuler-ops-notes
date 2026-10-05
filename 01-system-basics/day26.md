# day26 · 日志体系与轮转（journald 持久化 / logrotate / 调度者悬案）

> 主线：排查链最后一环——证据从哪来、存多久、怎么不撑爆盘。openEuler 24.03 是 **journald + rsyslog 双轨制**。

## 一、家底侦察

- `ls -ld /var/log/journal` → No such file = journald 跑成 volatile（重启后 journalctl 查不到上次开机）
- `du -sh /run/log/journal` = 8.0M；journald.conf 基本全默认（仅 ForwardToWall=no）
- rsyslog active（8.2312.0-8）写 /var/log/messages、secure、cron——文本日志一直有 logrotate 保着
- chronyd active + synchronized yes：日志时间线可信是多机对账前提

## 二、journald 持久化（四连翻车结案）

```bash
sudo mkdir -p /var/log/journal                            # ① 裸 mkdir 建骨架
sudo systemd-tmpfiles --create --prefix /var/log/journal  # ② 补妆（2755 root:systemd-journal + SELinux 标签）
sudo tee /etc/systemd/journald.conf.d/99-persistent.conf <<'EOF'
[Journal]
Storage=persistent
SystemMaxUse=200M
SystemKeepFree=1G
MaxRetentionSec=4week
EOF
sudo reboot   # ③ 整机重启才激活
```

- 四连翻车：AI 剧本吞 mkdir / tmpfiles 规则全是 z 只修不建 / persistent 不自建目录 / 首次 reboot 空账（旧 boot 没写过磁盘，-b -1 自然查不到）
- **切换只在开机 flush 路径发生，systemctl restart 不重读 Storage**（journald 自白：Runtime Journal → flushing to /var/log/journal → System Journal max 200.0M）
- 结案铁证：`--list-boots` 两条 + `-b -1 -n 5` 读出旧 boot 临终日志

## 三、logrotate 实战

```bash
sudo logrotate -d /etc/logrotate.d/lab-app   # 干跑只打印
sudo logrotate /etc/logrotate.d/lab-app      # 真跑：19M → 0 新本 + .1.gz 788K（24:1）
```

- size 判词正反两面：`log needs rotating` / `log size is below the 'size' threshold`
- ★跑片段 ≠ 跑主配置：片段路线不读全局 dateext → 后缀 `.1`；timer 走主配置才有 `-20261005`
- 台账 = /var/lib/logrotate/logrotate.status

## 四、rsyslog 片段开膛

- rotate 30（覆盖主配置 rotate 4）+ size +4096k（超 4M 才轮）+ maxage 365 + copytruncate
- copytruncate 原地截断 inode 不变、rsyslog 无感（小丢窗）；create 必须配 postrotate HUP 否则日志写进旧文件

## 五、调度者悬案（两个预判被打脸）

- 排除：logrotate.timer 不存在、/etc/crontab 全注释、root crontab 空
- dailyjobs 是备胎：`[ ! -f /etc/cron.hourly/0anacron ] &&` 守卫天天短路——crond 记 CMD ≠ 真干活
- ★anacron 二进制在 /usr/sbin/anacron（cronie 主包自带）——rpm -q 子包名查不到 ≠ 功能缺失
- 真链：crond → 0hourly(:01) → 0anacron（防重跑 + 电池检查）→ anacron -s → anacrontab → cron.daily
- 轮转日期稀拉 = anacron 每日跑 × size 阈值双条件，logrotate.status 显示 messages 上次 2026-10-3-3:51:1

## 六、journalctl 检索

```bash
sudo journalctl --since "08:00" -p warning --no-pager | tail -8
sudo journalctl _COMM=sudo --no-pager -n 5      # 按进程名
sudo journalctl -t logrotate --no-pager | tail -3   # 阴性证据 = 无警情
sudo journalctl --disk-usage                    # 20.5M / 200M 闸门
```

## 七、坑位速查

| 坑 | 正解 |
|---|---|
| volatile 丢日志 | 四步：mkdir → tmpfiles 补妆 → drop-in → 重启整机 |
| restart 不切 Storage | 只在 boot flush 路径激活 |
| 跑片段无日期后缀 | 片段不读主配置全局项 |
| copytruncate vs create+HUP | 按丢不丢得起选 |
| rpm -q 查不到包名 | ls 二进制验证 |
| crond 记 CMD ≠ 干活 | 看下游执行日志 |
| sudo + glob 在用户 shell 展开 | 显式路径 / sudo sh -c |

## 面试一句话

> "日志体系是两条轨 + 两套清仓规则：journald 结构化自管（persistent 只在开机 flush 激活），rsyslog 文本靠 logrotate（copytruncate vs create+HUP 按丢不丢得起选）。'日志为什么没轮'先查调度者（这台是 crond→0hourly→0anacron→anacron 补跑链，dailyjobs 是带守卫备胎）再查 size gate。我踩过的坑：rpm 查不到包名不等于功能缺失、crond 记了 CMD 不代表真跑了——排查永远以执行日志和回显为准，不以配置和假设为准。"
