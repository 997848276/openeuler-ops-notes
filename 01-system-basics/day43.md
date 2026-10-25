# day43 · 性能排查四账本（上）—— load / CPU / 内存（2026-10-25 · 128 oe-base）

> 本节对应手册 `openEuler命令解读与类比.md` 的 **§五十（上）**。

---

## 一、开机基线（自检）

```bash
date; uptime; who
chronyc tracking | head -6
df -h / /data; echo "---"; df -i /
systemctl is-active isulad chronyd firewalld
```

**实测回显要点**：

```
11:11:55 up 1 min, 1 user, load average: 1.42, 0.48, 0.17

Reference ID  : 6FE6BDAE (111.230.189.174)
Stratum       : 3
System time   : 0.000000000 seconds slow of NTP time
Last offset   : -0.106312186 seconds
RMS offset    : 0.106312186 seconds

/dev/mapper/openeuler_bogon-root  64G  6.3G  54G  11%  /
/dev/mapper/data_vg-data_lv       25G  522M  25G   3%  /data

isulad / chronyd / firewalld → 全 active
```

**两个细节**：

| 现象 | 判读 |
|---|---|
| `up 1 min` | 开机不足 1 分钟，load 三数尚在累积期 |
| `System time = 0` 但 `RMS offset = 0.106s` | **当前是准的**；106ms 是开机时 chronyd 一次性纠正的**残影** |

> ★ 判据：**`System time` 才是准不准，`Last/RMS offset` 是历史痕迹。**

---

## 二、先立分母（打脸 #49）

```bash
nproc                              # → 2
grep -c ^processor /proc/cpuinfo   # → 2
```

**我默认这台机是 4 核（把 254 的配置记到 128 头上），于是让起了 4 个压测进程 → 在 2 核机上超载 2 倍。**

> ★★ **纪律：凡是要拿"核数"当分母的判据，必须先 `nproc` 实测，不许凭记忆。**
> 与"NodePort 不能凭记忆"是同一类错——**跨机器、跨轮次的状态一律实测。**

---

## 三、load average 三个数怎么读

```
11:11:55 up 1 min, 1 user, load average: 1.42, 0.48, 0.17
```

| 数值 | 时间常数 | 用途 |
|---|---|---|
| 1 分钟 | 60 s | 瞬时压力 |
| 5 分钟 | 300 s | 中短期趋势 |
| **15 分钟** | 900 s | **长期基线（告警用它）** |

### 3.1 ★ 开机时三数必然倒挂

`1.42 > 0.48 > 0.17`，而机器 `up 1 min`。

**三个平均值都从内核启动那一刻的 0 开始累积**：

- 1 分钟值：已有 1 分钟真实样本 → 反映"开机风暴"（systemd 拉几十个服务）
- 5 分钟值：只有 1/5 权重，被大量 0 稀释
- 15 分钟值：只有 1/15 权重，几乎全是启动前的 0

> ★★ **开机 15 分钟内，load 三个数没有历史可比性。**
> "1min > 5min > 15min"是**数学必然**，不是"负载在恶化"。
> **生产启示**：监控要配「开机后 N 分钟内不判 load」，否则每次重启都假告警。

🎯 **类比**：刚开店 1 分钟问"今天生意如何"——1 分钟数据是真的，5 分钟平均被"还没开门"稀释，15 分钟几乎全是打烊时段。**不是生意变好，是样本不够。**

### 3.2 ★★ 指数平均的滞后性（本节最重要的认知）

**实测四组数**：

| 场景 | 时刻 | load 1min |
|---|---|---|
| 4 个负载进程跑了 30 秒 | 11:14:20 | **2.24**（不是 4.0） |
| 同一批跑了 41 秒 | 11:14:31 | **2.58**（还在爬） |
| kill 后 3 秒 | 11:15:08 | **2.39**（还没降） |
| kill 后约 80 秒 | 11:16:27 | 0.68 |

**两个推论**：

1. **升上去需要时间**：30 秒只爬到理论值约 40%，要 1~2 分钟才接近真值
2. **降下来更慢**：进程死了，load 还挂着——**它记得"刚才有一堆活儿在排队"**

🎯 **类比**：load 是**水温计**，不是**火焰开关**。火开到最大，温度计也得慢慢爬；火关了，水也不会瞬间凉。

> ★ **无意义的判据**：`kill` 完立刻看 load 判断"杀没杀掉" → 必然误判。
> **正确判据**：`pgrep -a <进程名>` 看残留；load 只用于看趋势。

### 3.3 load 必须带分母

| 经验阈值（N = 核数） | 含义 |
|---|---|
| `load < N × 0.7` | 健康 |
| `N × 0.7 ~ N × 1.0` | 关注 |
| `load > N × 1.0` | 排队，告警 |
| `load > N × 2` | 严重，请求开始超时 |

> ★ **告警建议用 `load15 > N`**：`load1` 抖动大易误报，`load15` 更能代表"持续压力"。

---

## 四、`top` 的 %Cpu 行

```
%Cpu(s): 41.4 us, 58.6 sy, 0.0 ni, 0.0 id, 0.0 wa, 0.0 hi, 0.0 si, 0.0 st
```

| 字段 | 含义 | 高了说明 |
|---|---|---|
| `us` | 用户态 | 应用在计算 |
| **`sy`** | **内核态** | **系统调用 / 中断多** |
| `ni` | nice 过的进程 | 低优先级任务多 |
| `id` | 空闲 | —— |
| **`wa`** | **等 IO** | **磁盘慢！加 CPU 无用** |
| `hi` | 硬中断 | 网卡/磁盘中断风暴 |
| `si` | 软中断 | **网络流量大**（收包处理） |
| `st` | 被虚拟化偷走 | 虚拟机被宿主挤占 |

---

## 五、`vmstat` 与 ★ 列对齐陷阱（打脸 #50）

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 2349540 26244 515880   0    0     0    12 2068  192  91  9  0  0  0
```

| 列 | 含义 | 判读 |
|---|---|---|
| `r` | 可运行 + 等 CPU 的进程数 | `r > 核数` = CPU 瓶颈 |
| `b` | 等 IO 的进程数（D 状态） | `b > 0` = IO 瓶颈 |
| `swpd` | 已用 swap（KB） | 持续增长 = 内存不足 |
| `free` | 空闲内存（**不含 cache**） | 小是正常的 |
| **`si` / `so`** | **换入 / 换出速率** | **非 0 = 正在 swap，严重** |
| `bi` / `bo` | 块设备读/写 | 高 = IO 压力 |
| `in` | 中断次数 | 高 = 中断风暴 |
| `cs` | **上下文切换** | 高 = 锁竞争 / 线程过多 |

### ★★ 我在这里犯的错

**我把 `2349540` 读成了 `swpd`，以为"swap 用了 2.3G"**，而 `free -m` 明明显示 `Swap used = 0` —— 凭空造出一个假警报。

**用 `awk` 一取就清楚了**：

```bash
vmstat 1 1 | tail -1 | awk '{printf "r=%s b=%s swpd=%s free=%s\n",$1,$2,$3,$4}'
# r=1 b=0 swpd=0 free=2349720
```

`swpd=0`，那个数是 `free` 列（2294 MiB）。

> ★★ **`vmstat` 是定宽表格，数字位数一变（`0` → `2349540`），肉眼对齐就错位。**
> **读数一律用 `awk '{print $N}'`，不要用眼睛。**
> 讽刺的是：刚写过"眼睛读表格会骗人"，十分钟后自己就犯。

🎯 **类比**：用肉眼对 Excel 的列 ≈ 用尺子在屏幕上量。看着差不多，实际差两位。

---

## 六、对照实验：`yes` vs `sha256sum`

### 6.1 `yes` 压测（4 个，2 核超载 2 倍）

```bash
rm -f /tmp/yes.pids
for i in 1 2 3 4; do yes > /dev/null & echo $! >> /tmp/yes.pids; done
sleep 30
uptime; top -bn1 | head -15; vmstat 1 5
xargs kill < /tmp/yes.pids; rm -f /tmp/yes.pids
pgrep -a yes || echo "无残留"
```

**回显**：

```
%Cpu(s): 41.4 us, 58.6 sy, 0.0 ni, 0.0 id, 0.0 wa
vmstat: r=4  us≈44  sy≈57  id=0  wa=0
load: 2.24 → 2.58（还在爬）
```

### 6.2 `sha256sum` 压测（2 个，正好 2 核）

```bash
rm -f /tmp/sha.pids
for i in 1 2; do sha256sum /dev/zero > /dev/null & echo $! >> /tmp/sha.pids; done
sleep 20
uptime; top -bn1 | head -4; vmstat 1 3
xargs kill < /tmp/sha.pids; rm -f /tmp/sha.pids
pgrep -a sha256sum || echo "无残留"
```

**回显**：

```
%Cpu(s): 91.3 us, 8.7 sy, 0.0 ni, 0.0 id, 0.0 wa
vmstat: r=2  us=91  sy=9  id=0  wa=0
```

### 6.3 对照表（本节核心）

| 负载方式 | us | sy | 说明 |
|---|---|---|---|
| 4 × `yes > /dev/null` | 41.4 | **58.6** | 每行一次 `write()` syscall，时间花在"进内核再出来" |
| 2 × `sha256sum /dev/zero` | **91.3** | 8.7 | 纯计算，syscall 只在读 4KB 块时发生 |

> ★★ **看到 `sy` 异常高，方向不是"算得太多"，而是"系统调用太多"**：
> 频繁读写小文件、频繁 `fork`、锁竞争、网络小包过多。**加 CPU 无效，要减少 syscall。**

🎯 **类比**：`sha256sum` 是一个人埋头算数学题（`us` 高）；
`yes` 是一个人在窗口反复交材料，每次都要递进去盖章拿回来（`sy` 高）。
**两人都"很忙"，但瓶颈一个在脑子、一个在窗口。**

### 6.4 压测工具选择

| 工具 | 压什么 | 特征 |
|---|---|---|
| `sha256sum /dev/zero` | **纯用户态计算** | us≈100，sy 极低 |
| `yes > /dev/null` | **系统调用** | us 中，**sy 高** |
| `dd if=/dev/zero of=/dev/null bs=1M` | 内存拷贝 + syscall | us/sy 混合 |
| `openssl speed` | 加密计算 | us 高 |

> ★ **`yes` 不是纯 CPU 压测工具**——它压的是 syscall 路径。

---

## 七、内存账本

```
               total   used   free   shared  buff/cache   available
Mem:            3372    766   2294        1         529        2605
Swap:           4095      0   4095
```

### 7.1 三处对质（`free` / `/proc/meminfo` / 换算）

| 指标 | `free -m` | `/proc/meminfo` | 核对 |
|---|---|---|---|
| 空闲 | `free 2294` | `MemFree 2349720 kB` | ÷1024 = 2294 ✅ |
| 可用 | `available 2605` | `MemAvailable 2668468 kB` | ÷1024 = 2606 ✅ |
| 缓存 | `buff/cache 529` | `Buffers 26348 + Cached 477068 + SReclaimable 38928 = 542344 kB` | = 529.6 ✅ |
| 交换 | `Swap used 0` | `SwapTotal = SwapFree` | 差 0 ✅ |

```bash
awk '/^Buffers|^Cached|^SReclaimable/{s+=$2} END{printf "buff/cache = %d kB = %.1f MiB\n", s, s/1024}' /proc/meminfo
# 实测：542392 kB = 529.7 MiB（与 free 的 529 对上）
```

> ★ **`buff/cache = Buffers + Cached + SReclaimable`**，不是只有 Cached。

### 7.2 ★ `used` 到底是什么（打脸 #51）

```
total - free - buff/cache = 3372 - 2294 - 529 = 549    ← 不符
total - available         = 3372 - 2605     = 767    ← ✅ 对得上 766
```

**先验证"进程吃多少"**：

```bash
ps -eo rss --no-headers | awk '{s+=$1} END{printf "所有进程 RSS 合计 = %.0f MiB\n", s/1024}'
# 实测：628 MiB
```

**于是 766 的构成清楚了**：

```
used 766
  ├─ 进程 RSS 合计      628   (82%)
  └─ 内核自留          ≈138   ← SUnreclaim 95 + 页表 + 内核栈
                合计    766  ✅
```

> ★★ **修正一个常见误解**：不能简单说"`used` 不是进程占用"。
> 准确说法是——**`used` 的主体是进程，但额外包含内核自留；
> 而"进程 RSS 合计"本身又会虚高**。两者偏差方向相反。

**我原来的猜测是"进程 RSS 只有 100~200 MiB"，实测 628 —— 错在忘了这台机上跑着 iSula 里的两个容器。**

### 7.3 ★ RSS 会虚高：共享页被重复计算

| 指标 | 算法 | 特点 |
|---|---|---|
| **RSS** | 进程映射的物理页 | **共享页每个进程都算一份** |
| **PSS** | 共享页**按进程数均摊** | 更接近真实占用 |

**举例**：`libc.so` 占 2 MB，被 20 个进程映射：
- RSS 合计 = 20 × 2 = **40 MB**（虚高 19 倍）
- PSS 合计 = **2 MB**

```bash
sudo awk '/^Pss:/{s+=$2} END{printf "PSS 合计 = %.0f MiB\n", s/1024}' /proc/[0-9]*/smaps_rollup 2>/dev/null
```

> ★ **`ps` 的 RSS 列不能直接相加当"总内存占用"**——共享库和共享内存会被重复计数。
> 要精确对比用 **PSS**，或直接用 `smem`。

### 7.4 进程排行（打脸 #52：我猜错了）

```bash
ps -eo pid,rss,comm --sort=-rss | head -8
```

```
  PID   RSS COMMAND
 1229 148124 python3          ← 148 MiB，占 RSS 总分 24%
    1  64368 systemd
 1175  44668 firewalld
 1241  36256 isulad
 1129  32044 restorecond
 1744  31924 zabbix_agent2
 1246  30896 tuned
```

**我预测 TOP3 是 isulad / zabbix_agent2 / node_exporter —— 全错。**
实际第一名是一个 **148 MiB 的 `python3`**（身份未定：用 `ps -p 1229 -o args=` 可查其命令行）。

### 7.5 唯一的内存判据

| 数 | 能当判据吗 |
|---|---|
| `free` | ❌ 没用（Linux 主动吃满做缓存） |
| `buff/cache` | ❌ 不是浪费，是好事 |
| `used` | ⚠️ 参考 |
| **`available`** | ✅ **唯一判据** |

> ★★ **`free` 小是正常的，`available` 小才是危险。**
> `available` 逼近 0 才会触发 **OOM Killer**；`free` 剩 200 MiB 完全不用慌。

---

## 八、三个隐藏指标

```
Dirty:       0 kB
Writeback:   0 kB
Slab:    136220 kB   ← SReclaimable 38928 + SUnreclaim 97292
```

| 指标 | 含义 | 高了说明 |
|---|---|---|
| `Dirty` | 改了还没写盘的页 | 写操作在攒着，回写高峰要来 |
| `Writeback` | 正在写盘的页 | 磁盘正被追着跑 |
| `SReclaimable` | 可回收（dentry/inode） | 能吐出来，不用管 |
| **`SUnreclaim`** | **不可回收** | **持续单调增长 = 内核态内存泄漏** |

> ★★ **`Dirty` 大 + `Writeback` 大 + `wa` 高 = "写盘追不上应用"**，数据库服务器最典型的一类故障。
> ★★ **`SUnreclaim` 持续增长不回落** → 驱动 / conntrack 表 / 内核模块泄漏，**重启进程无用**。

---

## 九、判据树

```
load 高（先除 nproc）
  │
  ├─ us 高 + sy 低 + wa≈0        → 纯计算瓶颈      → 优化算法 / 加核
  ├─ us 中 + sy 高 + wa≈0        → 系统调用过多    → 减少 syscall（cs 也高）
  ├─ wa 高 + vmstat 的 b 大      → 磁盘 IO 卡住    → 查盘 / 慢 SQL，加核无用
  └─ us、sy、wa 都不高但 load 高  → D 状态进程      → 查 NFS / 驱动 / 内核

内存
  ├─ available 充足        → 健康（free 小无所谓）
  ├─ available 逼近 0      → 要 OOM 了
  └─ si/so 非 0            → 已经在 swap，性能断崖
```

---

## 十、坑位速查

| # | 现象 | 正解 |
|---|---|---|
| 1 | 开机后 load 三数倒挂 | 平均值从 0 累积，**开机 15 分钟内无参考价值** |
| 2 | 压测 30 秒 load 只到一半 | 指数平均爬升需 1~2 分钟 |
| 3 | `kill` 完 load 不降 | 平均有记忆，**用 `pgrep` 判进程** |
| 4 | 2 核机 load 4 以为"半载" | **忘除分母**，`nproc` 必须先查 |
| 5 | `sy` 比 `us` 还高 | 系统调用太多，不是算得多 |
| 6 | `vmstat` 读出"swap 用了 2.3G" | **列错位**，用 `awk` 取字段 |
| 7 | `free` 的 `used` ≠ `total-free-cache` | 新版 `used ≈ total − available` |
| 8 | `free` 只剩 200MB 就慌 | 看 `available` |
| 9 | `free` 的 cache 与 `Cached` 对不上 | `buff/cache = Buffers + Cached + SReclaimable` |
| 10 | `ps` 的 RSS 相加当"总占用" | 共享页重复计算，用 **PSS** |
| 11 | `SUnreclaim` 一直涨 | 内核内存泄漏，重启进程无用 |

---

## 十一、打脸台账（4 条）

| # | 错误判断 | 实测真相 |
|---|---|---|
| 49 | 默认 128 是 4 核 | **2 核**，导致压测超载 2 倍 |
| 50 | `vmstat` 读出"swap 用了 2.3G" | 列错位，`swpd=0` |
| 51 | 进程 RSS 只有 100~200 MiB | **628 MiB**（忽略了 iSula 里的容器） |
| 52 | TOP3 = isulad/zabbix/node_exporter | **python3 148 MiB 排第一** |

**共同点**：**全是拿记忆里的数字套现场。四次预测全错，只有实测回显是对的。**

> ★ 定位方法：`ps -p 1229 -o pid,args` —— 先看 `args` 再断言进程身份。

---

## 十二、一句话总结

> 「看负载第一步先 `nproc`——load 是排队长度，不带分母没意义，2 核机的 4.0 是超载 100%，8 核机才半载。第二步拆 CPU 账：`us` 高是应用在算，`sy` 高是系统调用太多，`wa` 高是磁盘拖后腿——三种优化方向完全不同，`wa` 高时加 CPU 一点用没有。第三步看内存，`free` 小不是问题，Linux 会主动用内存做缓存，唯一判据是 `available` 逼近 0 才会 OOM。另外 load 是指数平均、有滞后：压测三十秒它才爬到四成，进程杀掉它还要挂一两分钟才降，所以判进程死没死只能靠 `pgrep`，不能靠 load。」

---

## 附：本篇覆盖范围

- **已覆盖**：load 三数 / `top` 的 `%Cpu` / `vmstat` 逐列 / 内存账本（`free`、`meminfo`、RSS↔PSS）
- **未覆盖**：IO 账本（`iostat` / `iotop` / `Dirty` 与 `wa` 的因果）、网络账本（`ss` / `sar -n`）→ 属 §五十（下）


