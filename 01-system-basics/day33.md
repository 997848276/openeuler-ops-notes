# day33 · 定时任务：cron 与 systemd timer 对决（2026-10-12 · 254/128 双机）

## 一 侦察：双管家家底

```bash
crontab -l 2>&1; sudo crontab -l 2>&1
ls /etc/cron.d /etc/cron.daily /etc/cron.hourly 2>/dev/null
sudo systemctl list-timers --all --no-pager        # ★systemd 命令一律 sudo
ls /usr/lib/systemd/system/*.timer /etc/systemd/system/*.timer 2>/dev/null
```

| | 128 oe-base | 254 oe-docker |
|---|---|---|
| 用户区 timer | backup-app(02:00)、inspect(`*:0/2`) | db-backup(02:30)、backup-check(hourly) |
| cron 链 | cron.d={0hourly,dailyjobs,mdcheck,timezone.cron}；cron.daily={logrotate,man-db.cron,rpm}；cron.hourly=0anacron | 同款结构 |

- list-timers 三列必读：**NEXT（下次）/ LAST（上次）/ ACTIVATES（触发哪个 service）**；timer 与 service 同名配对，也可 `Unit=` 显式指定（inspect.timer 就是）。
- ★**悬案①**：devops 用不了 crontab——`/etc/cron.deny` 不存在，但 **`/etc/cron.allow` 存在且为空**。cronie 规则：**allow 一旦存在即白名单模式，名单外一律拒绝**；空 allow = 全体普通用户出局（root 例外）。
- ★**悬案②**：普通用户 `ls /etc/cron.d` 被拒——目录 **drwx------ root 700**，防刺探设计（不许看系统有哪些定时任务）。

## 二 Persistent 补课活体：LAST 卡在开机时刻之谜

```bash
sudo systemctl cat backup-app.timer
# OnCalendar=*-*-* 02:00:00 / Persistent=true
sudo journalctl -u backup-app.service --since today | head -6
# 07:50:42 Starting → 07:50:47 [backup] today=snap-2026-10-12 prev=snap-2026-10-11 → done → Finished
```

- 现象：128 backup-app 与 254 db-backup 的 LAST 都正好是今早开机时刻（07:50:42）。
- 结论：**昨晚 02:00/02:30 机器关着，今早开机 6 秒后管家补课**。254 备份孤本 5 份里 **3 份时间戳不是 02:30**（08_0851、09_0804、10_1226）——全是补跑产物，Persistent 已救过这条管道三次。
- ★**timer vs cron 第一记**：cron 错过就是错过；timer 写 `Persistent=true` 开机补跑。

## 三 真故障案：备份断了 43 小时（完整破案链）

**报案**（两条 journal 互相咬合）：
```
db-backup.service: Main process exited, code=exited, status=2/INVALIDARGUMENT
backup-watchdog[1106]: CRIT 备份过期: 最新备份 demo-2026-10-10_1226.sql.gz 已是 43h 前（阈值 26h）
```

**破案五步**：
1. 看脚本 `/usr/local/bin/backup-demo.sh`：五道关卡 = readiness 探测（mysqladmin ping ×30，未就绪 exit 3）→ mysqldump → “Dump completed” 签名校验（exit 1）→ 体积下限 1024B（exit 1）→ gzip + 7 天清理；`set -euo pipefail`。
2. 手动跑 **rc=2 却零输出**——脚本开头 `exec >> "$LOG" 2>&1`，**证词全在 `/var/log/backup-demo.log`**；journal 只记“他死了，凶器编号 2”。问案要去案发现场。
3. LOG 揭晓：`mysqldump: Got error: 1045: Access denied for user 'root'@'localhost' (using password: NO)`——**dump 行从没带 `-u`/`-p`**；探测循环每次都 `MySQL ready`，**MySQL 活着，死的只是没带钥匙的敲门方式**。
4. 断点定位：10/10 12:26 补跑成功（旧容器 socket 免密可连）→ 10/11 02:30 准点跑首次 1045（当时机器连续在线）→ **断点 = 10/10 下午我们重建 mysql8 的 3306 收口手术**。**变更之后没验证下游依赖。**
5. 修复（先备份再动刀）：

```bash
sudo cp /usr/local/bin/backup-demo.sh /usr/local/bin/backup-demo.sh.bak-$(date +%F)
sudo sed -i 's/mysqldump --single-transaction/mysqldump -uroot -pRoot@123 --single-transaction/' /usr/local/bin/backup-demo.sh
sudo /usr/local/bin/backup-demo.sh
sudo tail -5 /var/log/backup-demo.log      # OK: demo-2026-10-12_0808.sql.gz (764K) + done（顺带清了 7 天前孤本）
```

- mysql8 无罪：`Up 13 minutes`——重建时带的 `--restart unless-stopped` 开机自起立功。
- ★**watchdog 立功**：备份断了没人知道，是 hourly 的 backup-check 按门铃报的警——**备份没有监控等于没备份**。
- ★**timer 优势再添一条**：失败也进 journal（status=2 记得清清楚楚）；cron 失败只能翻邮件。
- ⚠️ 安全债：修复把密码明文写进脚本——临时可，**正确姿势 `--defaults-extra-file=/root/.backup.cnf`（600）放凭据**。

## 四 对决总表

| 维度 | cron（挂钟） | systemd timer（管家） |
|---|---|---|
| 记账 | 邮件 / 自己重定向 | **journal 全程记账**（含失败 status） |
| 错过补跑 | ❌ 错过即丢 | **`Persistent=true` 开机补** |
| 失败告警 | 无 | 服务 Failed 状态 + 配套 watchdog service |
| 控制力 | 无 | 依赖/资源/`RandomizedDelaySec` 错峰/`Unit=` 显式配对 |
| 语法 | 5 字段表达式 | `OnCalendar`（`*:0/2`/hourly/daily）+ `OnBootSec`/`OnUnitActiveSec` |

- ★结论：新任务一律优先 timer（备份类必带 `Persistent=true`），cron 只留给兼容和极简场景。

## 五 当天投产：把“粉笔线”刻成“刻刀线”（254）

```bash
sudo tee /etc/systemd/system/docker-user-guard.service >/dev/null <<'EOF'
[Unit]
Description=Restore DOCKER-USER guard for openGauss 5432
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'iptables -C DOCKER-USER -i ens33 -p tcp --dport 5432 -j DROP 2>/dev/null || iptables -I DOCKER-USER -i ens33 -p tcp --dport 5432 -j DROP'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload && sudo systemctl enable --now docker-user-guard.service
# 模拟故障自愈（等于模拟开机，不用真重启）
sudo iptables -D DOCKER-USER -i ens33 -p tcp --dport 5432 -j DROP
sudo systemctl restart docker-user-guard.service && sudo iptables -L DOCKER-USER -n --line-numbers
```

- ★**幂等写法**：`iptables -C ... || iptables -I ...`——重复执行不会堆规则（`-C` 存在即无事）。
- 再加 timer 每 10 分钟巡逻（`OnUnitActiveSec=10min` + `Unit=`）：**开机重放 + 定期点名**，与 backup-check 同一套防御哲学。
- 附：`ip6tables -L DOCKER-USER` 报 No chain → **docker 未启用 v6 面**，`:::5432` 是空头支票，IPv6 悬案销案。

## 六 坑位速查

| 坑 | 正解 |
|---|---|
| devops 用不了 crontab | /etc/cron.allow 存在即白名单；空 allow = 全员拒绝 |
| ls /etc/cron.d 被拒 | 目录 700 root，防刺探；用 sudo |
| timer 的 LAST 是开机时刻 | 疑 Persistent 补跑；journalctl -u 验证 |
| 服务 status=2 却没报错 | 脚本 `exec >>LOG` 吞输出；去 LOG 找案发记录 |
| mysqldump 1045 (using password: NO) | 没带凭据；短期 `-uroot -pXXX`，正解 defaults-extra-file+600 |
| 变更后备份悄悄坏掉 | 变更后必验下游依赖；备份必须配新鲜度监控 |
| 补跑的备份文件名时间戳不是计划时刻 | Persistent 正常行为；验新鲜度用 mtime 别匹配文件名 |
| sed 改脚本前没备份 | 先 `cp x.sh x.sh.bak-$(date +%F)` 再动刀（幂等可复位） |

## 面试一句话

> 「定时任务我用 systemd timer：它把调度和执行拆成 timer+service 两个单元，journal 全程记账、失败留痕，配 `Persistent=true` 错过的任务开机补跑，还能用 `RandomizedDelaySec` 错峰——cron 是到点就响的挂钟，timer 是会记账、会补课的管家。更重要的是运维习惯：备份必须配新鲜度监控，变更后必须验证下游依赖——我们真踩过：一次容器重建让备份脚本静默断了 43 小时，最后是 watchdog 报警、脚本自己的日志给出了凶器编号。」
