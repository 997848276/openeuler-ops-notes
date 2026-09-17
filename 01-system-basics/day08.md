# DAY4 · 存储与 LVM（2026-09-17）

> 机器：128（oe-base）｜devops 密钥登录 2222。
> 昨晚在 VMware 加了一块 20G 虚拟盘（20GB + 精简置备 + 单个文件）。
> 主线：**认盘 → 解剖 → 建盘 → 在线扩容 → 永久挂载**，两次翻车都成了教材。

---

## 一、块 0 · `lsblk` 认盘 + 意外发现

```
sda 100G
├─sda1   2M  part            ← BIOS boot（GPT 引导）
├─sda2   1G  part  /boot
└─sda3  99G  part            ← ★ 物理卷（PV）
   ├─openeuler_bogon-root  65G  lvm  /
   ├─openeuler_bogon-swap   4G  lvm  [SWAP]
   └─openeuler_bogon-home  30G  lvm  /home
sdb  20G  disk               ← 昨晚加的裸盘（无分区）
```

**两个关键发现**
1. **openEuler 默认安装就是 LVM** —— 这台机本身就在用：sda3 整块做 PV，划出 root/swap/home 三个 LV。学 LVM 不是学抽象，是解剖自家系统。
2. **65+4+30 = 99G = 池子见底（VFree=0）** —— 这就是"/ 满了怎么扩"的真实困境：lvextend 无米下锅。**要么加盘（昨晚已干），要么缩别的 LV**。

类比：**PV = 整块地捐给土地银行；VG = 银行的池子；LV = 从池子接水管划出来的地（可随时改大小）**。分区是砌死的墙，LV 是可移动的隔断。

---

## 二、块 1 · 解剖三连：`pvs` / `vgs` / `lvs`

```
$ sudo pvs      # PV 层：谁被捐了
  /dev/sda3  openeuler_bogon lvm2 a-- 98.99g  PFree 0
$ sudo vgs      # VG 层：池子多大、剩多少
  openeuler_bogon 1 3 0 wz--n- 98.99g  VFree 0
$ sudo lvs      # LV 层：划出来哪些地
  home/root/swap … Attr -wi-ao----（a=active o=open 挂载中）
```

- **`VFree`/`PFree` = 池子余粮**，扩容前必看
- **`<` 尖括号 = "略小于"**（后面为它付出了一学费）

---

## 三、块 2 · 从零建一套（sdb → /data）

```
sudo pvcreate /dev/sdb                     # 裸盘 → PV（捐地）
sudo vgcreate data_vg /dev/sdb             # 组池子
sudo lvcreate -n data_lv -L 18G data_vg    # ★ 故意只划 18G，留 2G 当扩容燃料
sudo mkfs.xfs /dev/data_vg/data_lv         # 格式化（blocks=4718592 = 18GiB/4096）
sudo mkdir -p /data
sudo mount /dev/data_vg/data_lv /data
df -h /data && lsblk
```

**两个坑**
- **新 XFS "空盘" df 显示 Used 385M** —— 不是丢数据，是 XFS 自带元数据（internal log 64M + reflink 账本）。**看磁盘认 Avail 列**。
- **一个设备两个名字**：`/dev/data_vg/data_lv` 和 `/dev/mapper/data_vg-data_lv` 是同一个 dm 设备的别名。

---

## 四、块 3 · 在线扩容大戏（本节灵魂）

**第一幕**：`sudo fallocate -l 16G /data/bigfile` → df 92%，事故就位。

**第二幕（翻车）**：

```
$ sudo lvextend -r -L +2G /dev/data_vg/data_lv
  Insufficient free space: 512 extents needed, but only 511 available
```

**三条硬知识（报错换来的）**
1. **LVM 按 extent 分配（默认 4MiB）**：+2G = 512 格
2. **PV 可用 ≠ 磁盘标称**：PV 开头 ~1MiB 是 LVM 元数据 → 20GiB 只剩 5119 格，LV 用 4608，剩 511 —— 差 1 个
3. **`vgs` 里的 `<2.00g` 尖括号**早就在喊"我不到 2G"——按整数算必踩坑

**第三幕（修复，生产姿势）**：

```
$ sudo lvextend -r -l +100%FREE /dev/data_vg/data_lv
  Size of logical volume data_vg/data_lv changed from 18.00 GiB (4608 extents)
    to <20.00 GiB (5119 extents).
  File system xfs found on data_vg/data_lv mounted at /data.
  xfs_growfs /dev/data_vg/data_lv            ← "-r" 自动调的就是它
  data blocks changed from 4718592 to 5241856
  Logical volume data_vg/data_lv successfully resized.
```

- **`-r` 的工作流**：探测文件系统类型+挂载点 → XFS 调 `xfs_growfs`（**必须挂载态**）→ LV+FS 一次扩完
- **数字闭环**：4718592×4096 = 18GiB（与 mkfs 时同数）；5241856×4096 = <20GiB；df Use% 92%→83%
- **★★ 最值钱的坑**：**`lvextend` 不带 `-r` = 只扩 LV 不扩文件系统** —— df 纹丝不动，十有八九误判"扩容失败"。（类比：只换粗水管、不换大量程水表。）
- **XFS 只能放大、不能缩小**（且扩容必须在线挂载态）→ 划 LV 宁小勿大
- **生产纪律**：吃剩余空间**永远用 `-l +100%FREE`**，别用精确的 `-L +NG`

---

## 五、块 4 · fstab 永久挂载（黄金验证法立功）

```
sudo rm /data/bigfile                       # 清演习道具
sudo cp -a /etc/fstab /etc/fstab.bak.20260917   # ★ fstab 写错=开不了机，先备份
sudo blkid /dev/data_vg/data_lv             # 拿 UUID（设备名会变，UUID 不变）
echo '<真实UUID>  /data  xfs  defaults  0 0' | sudo tee -a /etc/fstab
sudo umount /data
sudo mount -a          # 黄金验证法：按 fstab 全量重挂，无输出=全对
df -h /data            # 挂回来了 = fstab 有效
```

**翻车实录（教科书级）**：AI 给的命令里 `UUID=xxx-xxx` 是占位符，被整条复制进 fstab → `mount -a` **当场抓获**：

```
mount: /data: can't find UUID=xxx-xxx.
```

**假如不验**：坏行潜伏到下次重启 → 开机卡 emergency mode。现在抓，5 秒。**"写完不验 = 没写"第三次实证**（`sshd -t` 验 ssh、`systemctl start` 验 unit、`mount -a` 验 fstab —— 同一条纪律三张面孔）。

**修复**：

```
sudo sed -i 's|^UUID=xxx-xxx|UUID=c953ac67-…|' /etc/fstab
tail -3 /etc/fstab          # 回读确认（"写进去 ≠ 写对了"）
sudo mount -a && df -h /data && lsblk    # 三绿收官：20G 3% /data
```

**fstab 六字段顺带讲透**（自家文件当教材）：
`UUID=… /data xfs defaults 0 0` = 设备 / 挂载点 / 类型 / 选项 / dump(0) / fsck 次序（/ 是 1 最先自检，普通盘 0）。
改完 fstab 顺手 `sudo systemctl daemon-reload`（systemd 会缓存 fstab 生成的挂载单元）。

---

## 六、踩坑清单（8 条）

1. 新 XFS"空盘"Used 385M = 元数据，看 Avail 别看 Used
2. `-L +NG` 差一个 extent（PV 元数据 ~1MiB）→ 吃剩余用 `-l +100%FREE`
3. `vgs/lvs` 的 `<` 尖括号 = 略小于，条件反射
4. `lvextend` 不带 `-r` = 只换管子不换水表（df 不动）
5. XFS 不能缩小、只能在线扩 → 划 LV 宁小勿大
6. 带占位符的命令整条复制 = 占位符进配置文件
7. fstab 写错开不了机 → 先备份 + `umount && mount -a` 黄金验证法
8. 改 fstab 后 systemd 缓存旧版 → `daemon-reload`

## 七、面试话术（3 条）

1. **"业务盘满了怎么扩？"** → 先 `vgs` 看 VG 余粮：有，`lvextend -r -l +100%FREE` 在线扩（-r 自动调 xfs_growfs，业务不停机）；没有，加盘 `pvcreate` + `vgextend` 再扩。
2. **"为什么用 LVM 不直接分区？"** → 分区是砌死的墙，LV 是可移动隔断：在线扩容、跨盘汇聚、快照回滚都靠它。
3. **"fstab 怎么安全修改？"** → 先备份；写 UUID 不写设备名；写完 `umount` + `mount -a` 验证，fstab 写错开不了机。

## 八、加练 · `vgextend` 横向扩池（07:56）：池子见底的第二种解法

昨天 `-l +100%FREE` 把 20G 吃干，VFree 又归零——"池子见底"重演，这次**加一块地并进池子**：

```
echo "- - -" | sudo tee /sys/class/scsi_host/host*/scan   # 热加盘后扫总线（sdc 这才现身）
sudo pvcreate /dev/sdc
sudo vgextend data_vg /dev/sdc
sudo lvextend -r -L +5G /dev/data_vg/data_lv
```

**回显关键数字**
- `vgs`：data_vg `#PV 2`、VSize `29.99g`、VFree `<10.00g` → 扩后 `<5.00g`（VSize = 5119+2559 个 extent 闭环）
- `pvs`：`/dev/sdb <20.00g` + `/dev/sdc <10.00g` 同属 data_vg —— **盘不一样大也能并池**（要求一样大的是 RAID，不是 LVM）
- `lvextend`：`5119 → 6399 extents`（+1280 = 正好 +5G，**一次成功**）
- 字节闭环：`6552576 blocks × 4096 = 26,839,351,296 bytes` = 回显的 `<25.00 GiB`
- `df`：20G→25G；**Used 424M→522M —— 涨的 ~98M 是 XFS 给新空间建的新账本**（元数据，不是文件）
- `blkid`：UUID `c953ac67-…` **纹丝不动** → fstab 一字不改，`mount -a` 静默通过

**三条硬知识**
1. **热加盘后系统不自动认盘**：开机才扫 SCSI 总线；`- - -` = 通配所有 channel/target/LUN 重新报数（云盘热扩容同款动作）
2. **`-L +NG` 冤案平反**：昨天差 1 个 extent 不是 `-L` 的错，是余粮刚好卡边界；今天池里有粮，`+5G` 稳过 —— **判据在余粮，不在写法**
3. **扩容不改文件系统 UUID**：fstab 认身份证不认容量 → 扩容不动 fstab

**踩坑清单 +2（累计 10 条）**
9. 热加盘 `lsblk` 看不到 ≠ 加盘失败 → 手动扫 SCSI 总线
10. XFS 扩容后 `df` 的 Used 会涨一点（新空间元数据）→ 别误判"多了文件"

## 九、下一步

- 推送 `day08.md` 到 GitHub（`01-system-basics/day08.md`）
- 可选：LVM **快照**（`lvcreate -s`，改配置前先照相）
- 明天：攻略 DAY5 · systemd 编排
