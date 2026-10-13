# day32 · 磁盘与文件系统排障：从双台账到真砸真修（2026-10-11 · 128 oe-base）

## 一 侦察四件套：面积与格子分开看

```bash
df -h                 # 货架面积：/data = data_vg-data_lv 25G（3%）
df -i                 # 货架格子：/data Inodes 1310 万（IUse 1%）
lsblk -f              # 根=ext4、/data=xfs ← 双制并存
findmnt /data         # rw,relatime,seclabel,attr2,inode64,logbufs=8,noquota
sudo du -sh /data/* 2>/dev/null
```

- ★**两种满两种病**：空间满（面积用尽）与 inode 满（格子用尽），报错同句 `No space left on device`，**病根靠 df -h 与 df -i 分诊**。
- ★根是 ext4、/data 是 xfs → **修复工具分家**：xfs 用 `xfs_repair`，ext4 用 `fsck.ext4`，拿错扳手会把好盘修坏。
- 伏笔：df 说 522M、du 只数出 52K——差值是文件系统元数据（inode 表、日志区）；du 数目录树里的文件，df 数整盘的块。
- 记账悬案：NFS 行 `1G used / 45G avail / 26%` 与账面不成比例（Use% 公式掺保留块，且 NFS 数字来自服务端 statfs），待去 254 服务端 df 对照。

## 二 第一案：df 与 du 打架——“日志删了空间为什么不回来”

```bash
sudo dd if=/dev/zero of=/data/big.log bs=1M count=2000 status=none   # ⚠️ 必须 sudo
df -h /data | tail -1                          # 3% → 10%
sudo sh -c 'exec 3>>/data/big.log; sleep 3600' &   # 起一个不重启的服务抱着日志
sudo rm /data/big.log
ls /data/big.log                               # No such file（门牌摘了）
df -h /data | tail -1                          # ⭐ 依然 10%（块没还）
du -sh /data                                   # ⭐ 52K（du 却变小了）

sudo lsof +L1 /data
# sleep 63913 root 3w REG 253,3 2097152000 1 147 /data/big.log (deleted)
sudo truncate -s 0 /proc/63913/fd/3            # 在线回收，服务不用重启
df -h /data | tail -1                          # ⭐ 10% → 3%
sudo lsof +L1 /data | grep big.log
# sleep 63913 3w … 0 0 147 /data/big.log (deleted)  ← SIZE=0 袋子倒空 / NLINK=0 门牌已摘
```

- ★**rm 的语义不是“释放空间”，是“摘门牌”**：块能不能还看引用计数，不看目录里还有没有名字。
- ★生产正解：`lsof +L1` 找句柄 → `truncate -s 0 /proc/PID/fd/N`，服务不重启。
- 判据读法：lsof 一行三字段 = SIZE/OFF（袋子）、NLINK（门牌）、(deleted)（人还在屋里）。
- 附：ps 揭示 sudo 三层进程链 sudo → sudo-child → sleep；kill 孙进程后整个 job 收尸。

## 三 第二案：inode 耗尽——“面积空 61%，格子 100% 满”

```bash
sudo dd if=/dev/zero of=/data/inode-lab.img bs=1M count=500 status=none
sudo mkfs.xfs /data/inode-lab.img          # ⚠️ 拒格 <300M：must be larger than 300MB
sudo mount -o loop /data/inode-lab.img /mnt/inode-lab
df -h /mnt/inode-lab | tail -1             # /dev/loop0 436M（★核设备名）
df -i /mnt/inode-lab | tail -1             # Inodes 256000 ← 格子总数

sudo sh -c 'cd /mnt/inode-lab && i=1; while [ $i -le 999999 ]; do : >f$i.txt 2>/dev/null || { echo "第 $i 个时爆了"; break; }; i=$((i+1)); done'
# 爆点 f256030（≈256000 格 + 保留 inode）
df -h /mnt/inode-lab | tail -1             # 39% ← 面积还空
df -i /mnt/inode-lab | tail -1             # 100% ← 格子全满
sudo touch /mnt/inode-lab/new.txt          # ⭐ No space left on device

sudo umount /mnt/inode-lab && sudo rm -rf /mnt/inode-lab /data/inode-lab.img
```

- ★**格子总数是格式化政策定的**：mkfs 回显 `imaxpct=25` = 最多 25% 面积给 inode 区 ÷ 512B ≈ 25.6 万格。小件多的场景（邮件、缓存）要提前调 imaxpct。
- ★**inode 本尊也占面积**：25 万 × 512B ≈ 128M——格子不是虚账，是磁盘上真实的元数据。
- ⚠️ **翻车教学**：第一版 100M 镜像 mkfs 失败 → mount 报 bad superblock → **df 静默降级显示根盘 64G**（df 只回答“该路径当下住在哪个 FS”，不回答“你以为它在哪”）→ 20 万文件全部错砸根盘（IUsed 97097→297100，正好 +200000 实锤）。**mount 后必须核 df 的设备名再干活。**
- ⚠️ 坑：`:` 是 special built-in，**重定向失败对它是致命的**，shell 当场死亡，那句 `|| echo` 和 break 都没跑。

## 四 第三案：损坏与修复——只读应急 + 真砸真修

```bash
# 应急止损：切只读（root 也写不进——内核挂载层锁，不是权限）
sudo mount -o remount,ro /data
sudo touch /data/try-write.txt     # ⭐ Read-only file system
sudo mount -o remount,rw /data

# loop 迷你盘：造文物 → 体检 → 破坏 → 修复 → 验收
sudo mkfs.xfs -q /data/repair-lab.img
sudo mount -o loop /data/repair-lab.img /mnt/repair-lab
sudo sh -c 'echo important-data-123 > /mnt/repair-lab/treasure.txt'
sudo umount /mnt/repair-lab
sudo xfs_repair -n /data/repair-lab.img | tail -3      # -n 干跑只查不修，Phase 1~7

sudo dd if=/dev/urandom of=/data/repair-lab.img bs=512 count=4 conv=notrunc status=none
sudo mount -o loop /data/repair-lab.img /mnt/repair-lab   # bad superblock
sudo xfs_repair /data/repair-lab.img | tail -5
# Phase 1: bad primary superblock → found candidate secondary superblock → writing modified primary superblock
sudo mount -o loop /data/repair-lab.img /mnt/repair-lab
cat /mnt/repair-lab/treasure.txt                      # ⭐ important-data-123 幸存
```

- ★**砸不掉的保险柜设计**：xfs 把盘切成几个 AG，**每个 AG 存一份全套图纸**（次级超级块），主 SB 毁了从任意 AG 抄回重建。
- ★为什么文物幸存：砸的是元数据（账本），数据块本体没碰——**修复=重建账本，不是恢复数据**。反之警惕：修不动的 inode 会被送 lost+found。
- ⚠️ **红线**：真实盘上 xfs_repair **必须先 umount**，挂载状态跑 = 亲手把好盘修坏；修之前能备份先备份。

## 五 坑位速查

| 坑 | 正解 |
|---|---|
| /data 里 dd 报 Permission denied | /data 属 root，命令带 sudo |
| 删了文件空间不回来 | 进程句柄还开着；lsof +L1 → truncate /proc/PID/fd/N |
| `sudo kill %1` 报 cannot find process | job spec 只归 shell 内建 kill；sudo 起外部 kill，用真 PID |
| `/proc/<PID>/fd/3` 报 PID: No such file | `<PID` 被当输入重定向；占位符必须换实查值 |
| df 显示的是根盘 | 挂载没成功时 df 静默报底层盘；mount 后必核设备名 |
| mkfs.xfs 报 must be larger than 300MB | 镜像加码到 500M |
| 盘没满却 No space left | `df -i` 看格子；inode 满了照样拒收 |
| `: >file` 失败后 break 没跑 | `:` 是 special built-in，重定向失败直接 fatal |
| root 也写不进文件系统 | remount,ro 的内核锁，非权限（应急止损用） |
| mount 报 bad superblock | xfs_repair（先 umount），它会自动用次级 SB 重建 |

## 面试一句话

> 「磁盘排障先分诊：`df -h` 看面积、`df -i` 看格子，报错都是 `No space left on device`，病根完全不同。空间满先查 `lsof +L1`——rm 只是摘门牌，块要等引用计数归零才还，生产正解是 truncate 句柄文件在线回收，不重启服务。格子满是格式化政策（imaxpct）决定的，小件多的系统要提前调。而 xfs 敢修，是因为它每个 AG 都存了全套图纸——超级块砸了从次级抄一份重建，数据块没坏数据就活着；但修复的底线是先卸载、能备份先备份，外科手术不该在病人还站着的时候开刀。」
