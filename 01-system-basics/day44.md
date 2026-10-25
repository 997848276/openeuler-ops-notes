# day44 · 性能排查四账本（下）—— IO 账本（2026-10-25 · 128 oe-base）

> 对应手册 `openEuler命令解读与类比.md` **§五十一**（IO 账本）。
> 本节所有数字来自 128 上的真机实验：4G 顺序写（`conv=fsync`）+ 2G 页缓存写。

---

## 一、先看清盘怎么拼的

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT | head -15
df -h /data
```

**实测**：

```
sda 1000G
├─sda2  1G  /boot
└─sda3 990G
  ├─openeuler_bogon-root  65G  /
  ├─openeuler_bogon-swap   4G  [SWAP]
  └─openeuler_bogon-home  30G  /home
sdb  20G ─┐
sdc  10G ─┴─ data_vg ── data_lv 25G  /data
```

**推理链**：`data_lv = 25G`，而 `sdb` 只有 `20G` → **25 > 20，LV 必然跨两块盘**（linear 模式：前 20G 落 sdb，后 5G 落 sdc）。

🎯 **类比**：两块地拼成院子，中间没墙也没备份。**面积相加，但塌一块整院作废。**

> 🚩 **风险**：LVM 线性拼接 = **容量叠加、零冗余**。sdb 或 sdc 坏任一块，`/data` 全丢。**和 RAID 是两件事**，最容易被混说成"做了 LVM 就有保护"。

---

## 二、工具盘点

```bash
for c in iostat pidstat sar iotop fio; do printf "%-8s %s\n" "$c" "$(command -v $c || echo 缺失)"; done
```

```
iostat   /usr/bin/iostat      ┐
pidstat  /usr/bin/pidstat     ├─ sysstat 包，默认已装
sar      /usr/bin/sar         ┘
iotop    缺失                 ← 默认没有
fio      /usr/bin/fio         ← 有，专业压测优先用它
```

| 工具 | 视角 | 用途 |
|---|---|---|
| `sar -d -p` | 设备，**列少** | 人眼读 |
| `iostat -x` | 设备，列多 | 脚本 / 精读 |
| `pidstat -d` | **进程** | 谁在写 |
| `fio` | 压测 | 测盘真实能力 |

> ★ `iostat -x` 与 `sar -d` 读的是**同一份 `/proc/diskstats`**，结论必须一致；不一致 = 读错列。

---

## 三、三个输出格式坑（全踩过）

### 3.1 `iostat -x` 窄终端折行

17 列放不下时，表头和数据各折成两行，**用眼睛对齐必错**。

### 3.2 `COLUMNS=200` 对 iostat 无效

```bash
COLUMNS=200 iostat -x 1 2      # 照样折行
```

**原因**：`iostat` 不读环境变量，用 `ioctl(TIOCGWINSZ)` 问终端**真实宽度**。
**命令没错，是假设错了。**

三条正解：

```bash
iostat -x 1 2 | cat                 # ★ 最省事：走管道就按固定宽度输出
iostat -x -o JSON 1 2               # 结构化，彻底绕开对齐
ip -o link                          # 同理：-o 一行一个，窄终端也安全
```

### 3.3 `vmstat -w` 只治数字、不治宽度

按位编号确认列数：

```bash
vmstat -w 1 1 | tail -1 | awk '{for(i=1;i<=NF;i++) printf "%d=%s ", i, $i; print ""}'
```

```
1=0 2=0 3=0 4=2051172 5=31160 6=796056 7=0 8=0 9=196 10=39 11=211 12=1 13=2 14=1 15=96 16=0 17=0 18=0
```

**18 列齐全**。位置表：

| 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | **16** | 17 | 18 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| r | b | swpd | free | buff | cache | si | so | bi | bo | in | cs | us | sy | id | **wa** | st | gu |

> ★★ **`wa` 永远是 `$16`**，`b` 是 `$2`。用 `awk` 取字段，不用眼睛数格子。

---

## 四、`iostat -x` 只认 6 列

| 列 | 含义 | 高了说明 |
|---|---|---|
| `r/s` `w/s` | 每秒 IO 次数 | 随机小 IO |
| `rkB/s` `wkB/s` | 每秒字节 | 吞吐大 |
| **`r_await` `w_await`** | **单次等待 ms** | **核心判据** |
| **`aqu-sz`** | 平均队列长度 | IO 版 load |
| `areq-sz` | 平均请求大小 | 小=随机，大=顺序合并 |
| `%util` | 设备忙占比 | **仅机械盘可信** |

---

## 五、4G 顺序写实测

```bash
rm -f /data/iotest/io-test.bin
( sar -d -p 1 14 > /tmp/sar-d.log 2>&1 ) &
( { time dd if=/dev/zero of=/data/iotest/io-test.bin bs=1M count=4096 conv=fsync status=none ; } 2>/tmp/dd.time ) &
for i in $(seq 10); do
  printf "%s " "$(date +%T)"
  awk '/^Dirty:/{printf "Dirty=%sMB ",$2/1024} /^Writeback:/{printf "WB=%sMB ",$2/1024}' /proc/meminfo
  sar -u 1 1 | tail -1 | awk '{printf "iowait=%s\n",$6}'
done
wait; cat /tmp/dd.time; ls -lh /data/iotest/io-test.bin
grep -E "DEV|sdb|sdc|data_lv" /tmp/sar-d.log | tail -14
```

**回显**：

```
real 0m8.657s   user 0m0.0013s   sys 0m6.679s
-rw-r--r--. 1 devops devops 4.0G  /data/iotest/io-test.bin

12:28:10 PM  tps 219.00  wkB/s 278528.00  areq-sz 1271.82  aqu-sz 4.69  await 21.40  %util 27.10  sdb
12:28:12 PM  tps 252.00  wkB/s 319488.00  areq-sz 1267.81  aqu-sz 7.42  await 29.42  %util 31.60  sdb
12:28:12 PM  tps   5.00  wkB/s 319488.00  areq-sz 63897.60 aqu-sz 0.32  await 63.60  %util 31.80  data_vg-data_lv
12:28:11 PM  tps   0.00  ...                                                        sdc
```

### 三个结论

**① 速度 = 473 MB/s**：`4 GiB / 8.657s`。

**② 写全落 sdb，sdc 一个字节没有** —— 实证 linear 拼接，4G ≪ 20G。

**③ `%util` 只有 30%，队列已排 7 个**

```
await  = 21.40 → 29.42 ms
aqu-sz = 4.69  → 7.42
%util  = 27.10 → 31.60     ← 看起来"很闲"
```

> 只看 `%util` 会得出"盘才用三成"的错判；实际每次 IO 等近 30ms，应用侧已经能感知。

---

## 六、LV 层 vs 物理层：`tps` 差 40 倍

```
sdb              tps = 219.00   areq-sz =  1271.82 kB
data_vg-data_lv  tps =   5.00   areq-sz = 55705.60 kB
```

同一批写：**LV 层看到 5 个大请求，物理层看到 219 个小请求**。

> ★★ 判 IO **只看物理设备**（sda/sdb/sdc）。映射层会合并请求，系统性低估 IOPS。

---

## 七、`Dirty` / `Writeback` 机理

### 7.1 阈值必须实测

```bash
sysctl vm.dirty_background_ratio vm.dirty_background_bytes vm.dirty_ratio vm.dirty_bytes \
       vm.dirty_expire_centisecs vm.dirty_writeback_centisecs
```

```
vm.dirty_background_ratio = 10        → 3372MB × 10% ≈ 337 MB 起后台回写
vm.dirty_background_bytes = 0         ← 0 = 未启用，ratio 生效
vm.dirty_ratio = 30                   → ≈1011 MB 应用被 throttle
vm.dirty_bytes = 0
vm.dirty_expire_centisecs = 3000      → 脏页超 30 秒强刷
vm.dirty_writeback_centisecs = 500    → 回写线程每 5 秒醒一次
```

> ⚠️ `*_bytes` 与 `*_ratio` **互斥，bytes 优先**。只改 ratio 不见效，先查 bytes。

### 7.2 两种速度，差 4 倍

| 写法 | 量 | 用时 | 速度 | 期间 `Dirty` | 测的是 |
|---|---|---|---|---|---|
| `dd ... conv=fsync` | 4G | 8.657s | **473 MB/s** | 44~52 kB | **磁盘** |
| `dd ...`（无 fsync） | 2G | ~1~2s | **1~2 GB/s** | **72.0 MB** | **内存** |

> ★★ **压测忘加 `conv=fsync`，速度虚高 4 倍。**

### 7.3 `Dirty` 是差额，不是总量

```
12:30:09.468 Dirty=72.0MB WB=0.0MB
...
=== sync 前 ===  Dirty=72.0MB
=== sync 后 ===  Dirty=0.0MB
```

**`Dirty` = 写入速率 − 回写速率**。数据库卡住时 `Dirty` 高，本质是**应用写入快过磁盘吞吐**。

只到 72MB 没到 337MB，因为：2G 总量不够顶阈值 + 后台回写吃掉大部分 + 采样窗只有 3.5 秒。

`Writeback` 全程 0.0 → **回写没积压**，只是"还没轮到"。

---

## 八、反直觉实测：`iowait = 0.00`

4G 写盘 8.657 秒里 `sys = 6.679s`，`iowait` **全程 0.00**。

**原因**：`iowait` 计的是「**CPU 空闲 且 有未完成 IO**」。这次 CPU 一直在内核态跑 syscall，**从未空闲**，所以计不到。

> ★★ **`iowait` 只在闲机器上灵敏，机器越忙越漏报。**
> 判 IO 压力：**`await` + `aqu-sz` ≫ `iowait`**。

---

## 九、环境坑四个（会让实验跑空）

### 9.1 `/data` 属主 root → Permission denied

```
drwxr-xr-x. 4 root root  /data
dd: failed to open '/data/io-test.bin': Permission denied
real 0m0.003s
```

**判 Permission denied 的顺序不能反**：

| ① DAC | `ls -ld /data` + `id` |
|---|---|
| ② 挂载 | `mount \| grep " /data "` |
| ③ SELinux | `ls -Zd /data` + `ausearch -m avc -ts recent` |

本机：`/data` others 无 `w` 位 → **DAC 就拒了**，与 SELinux 无关。
（标签确实是 `unlabeled_t`，但 `devops` 在 `unconfined_t` 域，绕过检查，无 AVC。）

**正解**：

```bash
sudo install -d -o devops -g devops -m 755 /data/iotest
touch /data/iotest/probe && echo "写权限 OK"
```

> ★ 别用 `chmod 777 /data` 图省事。给专用子目录 + 明确属主才是生产姿势。
> ★ `install -d` 幂等：已存在也不报错，只校正属主权限。

### 9.2 `/usr/bin/time` 不存在 → 退出码 127

```
-bash: /usr/bin/time: No such file or directory
[1]+ Exit 127
```

`time` 是独立包，默认没装。用 **bash 内建**：

```bash
( { time dd ... ; } 2>/tmp/dd.time ) &
```

| 码 | 含义 |
|---|---|
| 126 | 找到但不能执行 |
| **127** | **命令不存在** |
| 1 | 跑了但失败 |
| 137 | 128+9，被 KILL |

### 9.3 `rm -f` 的沉默

`-f` 的设计就是"文件不存在也不报错"。**它成功什么都证明不了。**

### 9.4 后台任务静默失败

`&` 不会告诉你"一启动就挂了"，只有 `wait` / `jobs` 看得到 `Exit N`。把 stderr 重定向进文件 = **把证据藏起来**，必须回头 `cat`。

> 实测教训：第一次实验 `Exit 127`、第二次 `Exit 1`，两次脚本都"跑完了"，但**一个字节都没写**，采样出来的 `Dirty=44kB` 全是空转数据。

---

## 十、判据树

```
wa 高 / 应用变慢
  ├─ await 高 + aqu-sz 高 + wkB/s 高  → 吞吐饱和    → 换盘/加盘/拆分
  ├─ await 高 + aqu-sz 高 + wkB/s 低  → 随机小 IO   → 合并写/调大 block
  ├─ await 低 + w/s 极高              → IOPS 打满   → 查元数据风暴
  ├─ Dirty 持续 > background          → 写积压      → 限流或换盘
  ├─ Writeback 常驻高                 → 回写被拖住  → 查慢盘/虚拟化限速
  └─ %util 高但 await 低              → 假饱和      → 忽略 %util
```

进程维度：

```bash
pidstat -d 1 5
pidstat -d -p <PID> 1 5
```

---

## 十一、坑位速查（13 条）

| # | 现象 | 正解 |
|---|---|---|
| 1 | 以为 `/data` 单盘 | `lsblk` 一看：跨 sdb+sdc |
| 2 | 以为 LVM 有冗余 | 线性拼接零冗余，坏一块全丢 |
| 3 | `iostat -x` 读数错位 | `\| cat` 或 `-o JSON` |
| 4 | `COLUMNS` 对 iostat 无效 | 它读终端真实宽度 |
| 5 | `vmstat` 少列 | 截断；`awk` 编号会看到 18 列 |
| 6 | `%util=30%` 以为很闲 | 看 `await`+`aqu-sz` |
| 7 | 看 LV 的 `tps` | 看物理设备 |
| 8 | `dd` 速度当磁盘速度 | 没 `conv=fsync` 测的是内存 |
| 9 | 调 ratio 无效 | `dirty_*_bytes` 优先 |
| 10 | `Dirty` 高以为内存不足 | 是写入>回写的差额 |
| 11 | 写盘时 `iowait=0` | CPU 不空闲时漏报 |
| 12 | `/usr/bin/time` 缺失 | 用内建 `time`；127=命令不存在 |
| 13 | 后台失败无感 | `wait` 查 `Exit N` |

---

## 十二、打脸台账（7 条）

| # | 错误判断 | 实测真相 |
|---|---|---|
| 53 | `/data` 在 sda 上 | **跨 sdb(20G)+sdc(10G)** |
| 54 | `fio` 缺失 | **在**；缺的是 `iotop` |
| 55 | `vmstat` 少了列 | **18 列齐全**，看折行看错了 |
| 56 | `Dirty` 会到 300~670MB | 带 fsync 只 **44~52 kB** |
| 57 | `iowait` 会到 20~60% | **全程 0.00** |
| 58 | `/data` 标签 `default_t` | **`unlabeled_t`** |
| 59 | `dirty_background_bytes` 非 0 | **= 0** |

**共同点**：七条全是拿"别处的默认值"套现场。**对的那两条（`%util<5%`、文件 4.0G）都是实测出来的。**

---

## 十三、一句话总结

> 「判 IO 第一步 `lsblk` 看盘怎么拼——这台 `/data` 是 20G+10G 拼的 25G LV，线性拼接没冗余，写先落第一块盘，坏一块全丢。第二步看 `await` 和 `aqu-sz`，不看 `%util`：实测写 4G 时 `%util` 只有 30%，但 `await` 已 21~29 毫秒、队列排到 7 个。第三步分清 `Dirty`：它是"写入速率减回写速率"的差额，阈值要先 `sysctl` 实测，`dirty_*_bytes` 会覆盖 ratio。第四步记住两个假象：`dd` 不加 `conv=fsync` 测的是内存不是盘，速度虚高四倍；写盘时 `iowait` 常常是 0，因为 CPU 从没空闲。`%util`、`iowait`、`Dirty` 三个数都会骗人，`await` 和 `aqu-sz` 不会。」
