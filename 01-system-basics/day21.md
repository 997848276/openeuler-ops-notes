# day-openeuler-0930 · LVM loop 沙盒重练（加练第一顺位）

> 日期：2026-09-30（周三）上午 · 机器 128 / oe-base（openEuler 24.03）
> 性质：加练（非主线 DAY），用 loop 设备从零沙盒重练 LVM，零风险不碰系统盘 sda
> 推 git：延后（等用户喊传，推前 `gops log` 确认最新序号 day20）

## 一、环境侦察（07:20）
- 128 三盘机：sda 100G（系统盘，openeuler_bogon VG：root 65G / swap 4G / home 30G）+ sdb 20G + sdc 10G
- sdb+sdc 已拼成 `data_vg`（29.99G）→ 切 `data_lv` 25G 挂 /data（跨盘拼池活教材，本次用 loop 从零重演）
- 坑：`mkdir /root/lvm-lab` → Permission denied；改 `~/lvm-lab`

## 二、第一课：loop 造盘 + 三件套（07:36）
- `dd if=/dev/zero of=diskN.img bs=1M count=200` ×2 → `sudo losetup -f --show` → loop0/loop1
- `pvcreate` → `vgcreate vg_lab`（392m）→ `lvcreate -L 250M -n lv_data`
- **PE Rounding 铁证**：250M → 实际 252.00 MiB（63 extents，4M/PE 向上取整）

## 三、撞坑：mkfs.xfs 拒绝 <300MB（07:36 → 修复 07:41）
- `mkfs.xfs` 报 `Filesystem must be larger than 300MB`（252M 不够门槛）
- 连锁：`mount` 报 `wrong fs type, bad superblock` = 设备无 fs（上游 mkfs 没成，下游背锅）
- 修复：`lvextend -L 320M`（63→80 extents）→ `mkfs.xfs` 成功 → mount
- **df≠LV**：320M LV 的 df 可用仅 256M（xfs 元数据 log 16M+AG 开销）；老数据 test.txt 写入成功

## 四、第二课：在线扩容（07:52）
- `dd` 100M big.bin 占位（Use% 48%）
- `dd` disk3.img 300M → loop2 → `vgextend vg_lab`（3PV，VFree 368m）
- `lvextend -L 600M`（80→150 extents）
- **`xfs_growfs /mnt/lvdata`**（参数是挂载点不是设备！对照 ext4 resize2fs 吃设备）
- data blocks 81920→153600；df 256M→536M（Use% 48%→24%）；test.txt 原封不动 = 在线无损闭环

## 五、第三课：快照 CoW（07:57 撞坑 → 00:01 修复）
- 撞坑：`lvcreate -s -L 100M` 报 `insufficient free space (22 extents): 25 required`
  → 快照本身也是 VG 里的特殊 LV（Attr 带 s），要占 extent；LV 600M 扩完 VFree 仅 22PE < 25PE
- 修复：`-l 100%FREE`（按 extent 单位用光剩余）；`mount -o ro` 只读挂
- 演示：原卷 `rm new.txt` → 原卷 ls 无、快照 ls 仍在、cat 内容原样 = CoW 写时复制实锤
- 报 `special device does not exist` 也是上游 lvcreate 失败连锁，先查 lvcreate

## 六、缩容认知
- **xfs 只扩不缩**（无缩小能力）；想缩容换 ext4（resize2fs，离线 + e2fsck）
- 生产缩容高危：正确姿势 = 备份 → 重建小 LV → 恢复，而非原地 lvreduce

## 七、清理
- `umount /mnt/lvsnap /mnt/lvdata`
- `lvremove -f vg_lab/lv_snap vg_lab/lv_data` → `vgremove vg_lab` → `pvremove /dev/loop0/1/2`
- `losetup -d /dev/loop0/1/2` → `rm -f ~/lvm-lab/*.img`

## 八、面试一句话
LVM 把"硬盘"和"分区"解耦（VG 池 / LV 逻辑盘），扩容在线不卸载（xfs_growfs 吃挂载点）、备份用 CoW 快照（lvcreate -s，记得它也要占池子空间）；xfs 只扩不缩是与 ext4 最大区别。
