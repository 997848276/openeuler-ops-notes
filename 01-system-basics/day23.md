# DAY23 · 备份与同步：rsync 增量 / 跨机免密 / NFS / --link-dest 快照 / timer 定时化（2026-10-02 · 128 oe-base + 254 oe-docker）

## 0 · 侦察

```bash
rpm -q rsync nfs-utils 2>&1; which rsync 2>&1        # 两件套在不在
df -hT | grep -Ev 'tmpfs|overlay'                    # 盘位：/data = data_vg-data_lv 25G xfs
ls -la ~/.ssh/ | head -5                             # 只有 authorized_keys（别人投的公钥），无私钥
ssh -o BatchMode=yes root@192.168.252.254 'hostname' # → Host key verification failed
getenforce                                           # Enforcing
```

- `Host key verification failed` = 第一次见 254（known_hosts 无档案）+ BatchMode 禁交互（被捆嘴）→ **不是断网**。
- authorized_keys 是别人投进来的公钥；**私钥才是自己口袋里的钥匙**。

## 1 · 装包 + 办门卡（免密通道）

```bash
sudo dnf install -y rsync nfs-utils                  # rsync-3.2.7-10 / nfs-utils-2.2.6.3-1（连带 quota、rpcbind）
systemctl list-unit-files --type=service | grep -E 'nfs|rpcbind'
ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519     # 空口令=教学便利；生产要口令短语+ssh-agent
ssh-copy-id root@192.168.252.254                     # yes 指纹 + 一次密码 → Number of key(s) added: 1
ssh -o BatchMode=yes root@192.168.252.254 'hostname; whoami'   # 不问密码直接 oe-docker / root
```

- unit 花名册：`nfs-server disabled`（待点火）/ `rpcbind enabled`（预装铺好软链）/ `nfs-mountd static`（**无 [Install] 段不能 enable**，被 Requires 拉人）/ `nfs` 等 **alias**。
- 主机指纹是服务器身份证：devops 与 root 看到同一枚 `SHA256:JyWWRDD…`，变的只是记没记过。

## 2 · rsync 本地三连：全量 → 增量 → 镜像

```bash
sudo rsync -av /data/app-data/ /data/backups/app-$(date +%F)/    # 首备 speedup 1.00
sudo rsync -av /data/app-data/ /data/backups/app-$(date +%F)/    # 重跑：空列表 speedup 4048.78
rsync -av --dry-run --delete /data/app-data/ /data/backups/app-$(date +%F)/  # 预演：deleting 行 + (DRY RUN)
rsync -av --delete /data/app-data/ /data/backups/app-$(date +%F)/            # 真删
```

- ★ 翻车 code 11：`mkkdir … failed: No such file or directory` —— **rsync 只建目标最后一级**，父级先 `mkdir -p`；彩蛋 `--mkpath`。
- ★ 增量量化铁证 = **speedup 三部曲 1.00 → 4048.78 → 2811.40**（speedup ≈ 总量 ÷ 实际传输量）。
- ★ 尾斜杠语义（第一大坑）：带 `/` = 搬内容；不带 = 连目录本身搬（多套一层）。
- 两本账：`du 12K`（磁盘块）≠ `total size 77`（内容字节）。

## 3 · 跨机推送 128 → 254

```bash
ssh root@192.168.252.254 'mkdir -p /backup'                       # 远端打地基
rsync -av /data/app-data/ root@192.168.252.254:/backup/app-$(date +%F)/
printf 'secret=Root@123' | sudo tee /data/app-data/conf/secret.ini >/dev/null && sudo chmod 600 $_
rsync -av /data/app-data/ root@…:/backup/app-$(date +%F)/        # → Permission denied (13) + code 23
sudo rsync -av -e "ssh -i /home/devops/.ssh/id_ed25519 -o UserKnownHostsFile=/home/devops/.ssh/known_hosts" \
  /data/app-data/ root@…:/backup/app-$(date +%F)/                 # 正解：显式借 devops 的钥匙+通讯录
sudo md5sum /data/app-data/conf/app.ini /data/app-data/conf/secret.ini /data/app-data/logs/app.log /data/app-data/upload/new.txt
ssh root@192.168.252.254 'cd /backup/app-2026-10-02 && md5sum conf/app.ini conf/secret.ini logs/app.log upload/new.txt'
```

- ★ code 12：`bash: line 1: rsync: command not found` —— **两端都要装 rsync**（搬运工双人舞）；`bash:` 前缀 = 远端在喊话。
- 错误码三兄弟：**11 = 文件 IO 错 / 12 = 协议数据流错 / 23 = 部分传输错**。
- 裸 sudo：root 找 `/root/.ssh`（没钥匙没档案）→ yes + 密码，能通但走密码；正解 `-e "ssh -i …"`。
- ★ 认证方式与数据同步是两本账：换钥匙重跑空列表（size+mtime 一致不重传）。
- 收官 = **双端 md5 对账逐行相同**；254 端 secret.ini `-rw------- root root`（-a 连属主权限都搬）。

## 4 · NFS：254 出房，128 租房

```bash
# 254 服务端
ssh root@192.168.252.254 'dnf install -y nfs-utils && systemctl enable --now nfs-server'
ssh root@192.168.252.254 'printf "/srv/nfsshare 192.168.252.128(rw,sync,no_subtree_check,root_squash)\n" > /etc/exports && exportfs -rav'
ssh root@192.168.252.254 'firewall-cmd --permanent --add-service={nfs,rpc-bind,mountd} && firewall-cmd --reload'
# 128 客户端
sudo mount -t nfs 192.168.252.254:/srv/nfsshare /mnt/nfs-254      # → type nfs4 vers=4.2 sec=sys
echo "written by 128 root" | sudo tee /mnt/nfs-254/from-128.txt   # 首次 → Permission denied（教学坑）
ssh root@192.168.252.254 'chown nobody:nobody /srv/nfsshare'      # 所有权交给访客代表
echo "written by 128 root" | sudo tee /mnt/nfs-254/from-128.txt   # → 成功，双端 ls 同显 nobody nobody
touch /mnt/nfs-254/devops-tried.txt                                # devops → 依然 denied（思考题）
```

- ★ root 写入被拒 = **两层门叠加**：①root_squash 把 uid 0 压成 nobody ②目录 `root:root 755`（nobody 无 w）。
- devops（uid 1000）**不被 squash**（root_squash 只逮 uid 0）但 755 的"其他人"只有 r-x → 照样 denied：**身份没被压、门禁永远看权限位**。
- SELinux：Enforcing 下实测不贴标签也通（`nfs_export_all_*` boolean 默认放行）。

## 5 · --link-dest 硬链接快照

```bash
sudo rsync -av /data/app-data/ /data/backups/snap-1/
printf '…night watch\n' | sudo tee -a /data/app-data/logs/app.log
sudo rsync -av --link-dest=/data/backups/snap-1 /data/app-data/ /data/backups/snap-2/   # 列表只传变化
sudo ls -li /data/backups/snap-{1,2}/conf/app.ini    # inode 同 67108994 + link count 2
sudo du -sh /data/backups/snap-1 /data/backups/snap-2 # 16K vs 4.0K（du 连跑去重）
```

- ★ inode 铁证：未变文件两代**同 inode + link count=2**；变的 inode 不同；字节账 total 92→96（改 4 字节涨 4）。
- 异地副本翻车：`chown … Operation not permitted (1)` + code 23 —— root_squash 剥夺 CAP_CHOWN → **`-a --no-o --no-g`**。
- 补作业机制：修复重跑全列重传（上轮 mtime 未归位）—— **rsync 账本 = size + mtime**。
- ★ 硬链接不出文件系统：link-dest 与目标必须同盘，跨 NFS 静默退化全量复制。
- 3-2-1 雏形：本地快照 + 254 /backup + NFS offsite = 3 份副本、2 个设备、1 份异地。

## 6 · fstab 固化 + timer 定时化

```bash
sudo cp /etc/fstab /etc/fstab.bak                   # 保命第一
printf '192.168.252.254:/srv/nfsshare  /mnt/nfs-254  nfs4  defaults,_netdev  0 0\n' | sudo tee -a /etc/fstab
sudo umount /mnt/nfs-254 && sudo mount -a && mount | grep nfs-254   # 拔了再插还活着
sudo systemctl daemon-reload                         # 让 systemd 认账
systemctl list-units --type=mount | grep -i nfs      # mnt-nfsx2d254.mount（- → \x2d 转义）
# /usr/local/bin/backup-app.sh：PREV=排除今日取最新代；本地失败 exit 1 / 异地失败 exit 2
# backup-app.service(Type=oneshot) + backup-app.timer(OnCalendar=*-*-* 02:00:00, Persistent=true)
sudo systemctl enable --now backup-app.timer         # → Created symlink timers.target.wants/…
sudo systemctl start backup-app.service              # 手动触发：status=0/SUCCESS + TriggeredBy: backup-app.timer
journalctl -u backup-app.service -n 15 --no-pager    # Starting → [backup] 行 → Finished
```

- ★ 真失败 vs 假失败：巡检"发现告警"exit 非 0 是**假失败**（unit 染红是冤案）；备份"rsync 没成"exit 1/2 是**真失败**（unit **就该红**才有告警）。

## 坑位速查（精选）

| 现象 | 正解 |
|---|---|
| code 11 `mkkdir … No such file` | rsync 只建最后一级 → 先建父级（或 --mkpath） |
| code 12 `rsync: command not found`（bash: 前缀） | 两端都要装 rsync |
| code 23 `Permission denied (13)` | 源文件无读权 → sudo 或接受部分失败 |
| 裸 sudo 跨机要密码 | `-e "ssh -i 钥匙 -o UserKnownHostsFile=通讯录"` |
| NFS root 写入 denied | root_squash×权限位两层门 → `chown nobody:nobody 导出目录` |
| NFS 上 chown `Operation not permitted` | `--no-o --no-g` 放弃属主搬运 |
| 修复重跑全列重传 | 上轮 mtime 未归位，账本=size+mtime，跑完即归位 |
| --link-dest 在 NFS 不省空间 | 硬链接不出文件系统，须同盘 |
| fstab 改完 systemd 不认 | `systemctl daemon-reload` |
| 开机挂 NFS 卡住 | fstab 漏 `_netdev` |

## 面试一句话

> 备份闭环：rsync 增量靠 speedup 量化、镜像靠 `--delete --dry-run` 预演；跨机两端都要装 rsync，sudo 用 `-e "ssh -i"` 借密钥，收尾双端 md5 对账；NFS 的 root_squash 说明身份映射和权限位是两道门；--link-dest 快照用 inode+link count 验证，只涨变化量；fstab `_netdev` 固化 + timer 每日 02:00 + Persistent 补跑，备份失败就该让 unit 变红——3-2-1 原则落地。
