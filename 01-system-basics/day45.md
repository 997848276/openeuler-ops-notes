# day45 · 性能排查四账本（终）—— 网络账本（2026-10-25 · 128 oe-base ↔ 254）

> 对应手册 `openEuler命令解读与类比.md` **§五十二**。
> 数据来自 128↔254 的 iperf3 实测（峰值 5.24 Gbit/s）+ `ss` / `nstat` / `sar -n DEV`。

---

## 一、网络账 = 四本小账

| 小账 | 看什么 | 命令 |
|---|---|---|
| 带宽 | 每秒字节、链路利用率 | `sar -n DEV 1 3` |
| 连接 | 连接数、状态分布 | `ss -s` / `ss -ant` |
| 丢包重传 | 协议栈计数器 | `nstat -az` |
| 错误 | 收发错误/丢弃 | `sar -n EDEV 1 3` |

> 只 `ping` 通不通，等于只看体温下诊断。

---

## 二、接口与基线

```bash
ip -br addr show
sar -n DEV 1 3 | tail -8
```

```
lo     UNKNOWN  127.0.0.1/8 ::1/128
ens33  UP       192.168.252.128/24 fe80::20c:29ff:fe39:52e2/64

Average:  ens33  5.33  6.67  0.50  4.94  0.00  0.00  0.00  0.00
```

| 列 | 含义 |
|---|---|
| `rxpck/s` `txpck/s` | 每秒包数（小包多 = PPS 压力） |
| **`rxkB/s` `txkB/s`** | **每秒千字节（不是 bit！）** |
| `rxmcst/s` | 组播 |
| `%ifutil` | 链路利用率（虚拟网卡不可信） |

### ★ 单位换算

```
Mbit/s = kB/s × 8 ÷ 1024
```

实测印证：

```
sar    rxkB/s = 680120.99
换算   = 680120.99 × 8 ÷ 1024 ≈ 5.19 Gbit/s
iperf3 = 5.24 Gbit/s
```

**两条独立命令互证**。把 kB/s 当 kbit/s 会把 5G 说成 0.6G，**差 8 倍**。

---

## 三、连接账：`ss -s`

```
Total: 145
TCP:   53 (estab 3, closed 39, orphaned 0, timewait 37)
```

| 字段 | 判读 |
|---|---|
| `estab` | 应用实时并发 |
| `closed` | 已关未清理，正常 |
| **`timewait`** | **本机是主动关闭方** |
| `orphaned` | >0 查泄漏 |

> `timewait` 多 ≠ 故障，真风险是**本地端口耗尽**。

---

## 四、三条 `established` 各是什么

```
192.168.252.128:1004    → 254:2049      NFS
192.168.252.128:2222    → .1:9402       SSH（宿主）
[::ffff:...]:9100       ← 254:49156     node_exporter 被抓取
```

**`Send-Q 148`** = 已发出**未被 ACK** 的字节，不是"发不出去"。

---

## 五、TIME_WAIT 解码（踩坑 + 正解）

### 5.1 踩坑

```bash
ss -tan state time-wait | awk '{print $5}' | sort | uniq -c | sort -rn | head -5
```

```
     35          ← 35 条空值
      1 Peer     ← 表头
```

`state time-wait` 过滤后**对端列不在第 5 列**。

> ★ `ss` 的列数随选项变（`-p` 加 Process，`-i` 加一堆 TCP 字段），**用位置取字段 = 把实现细节写死**。

### 5.2 正解：正则提取

```bash
ss -tan state time-wait | grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}:[0-9]+' \
  | grep -v '192.168.252.128:' | sort | uniq -c | sort -rn | head -5
```

### 5.3 实测判读

```
本地端口：56070 / 56062 / 56050 / 56016
```

**连续高位随机端口 → 本机是客户端，主动发起短连接**。

| 现象 | 结论 |
|---|---|
| 本地端口连续高位随机 | 本机**主动发起** |
| 本地端口固定业务端口 | 本机**被连**，TW 在对端 |
| 对端集中同一 IP:端口 | 某服务**周期性短连** |

---

## 六、协议栈总账：`nstat`

```
TcpRetransSegs            30     0.0
TcpOutRsts               120     0.0
TcpInErrs                  0     0.0
IpInDiscards               0     0.0
TcpExtTCPLost_Retransmit  10     0.0
```

### ★★ 两列含义完全不同

| 列 | 含义 |
|---|---|
| 第 1 列 | **开机累计**（判历史） |
| 第 2 列 | **每秒速率**（判现在） |

> 看到 `30` 就说"现在有 30 个重传"是误判。**当前看速率列**。

---

## 七、iperf3 打流：5.24 Gbit/s

```bash
# 128 服务端
iperf3 -s -1 > /tmp/iperf-s.log 2>&1 &
# 254 客户端
iperf3 -c 192.168.252.128 -t 10 -P 2
```

**实测（2 流 10 秒）**：

```
[SUM] 0.00-10.00 sec  6.10 GBytes  5.24 Gbits/sec  receiver
```

**实测（单流 8 秒）**：

```
[SUM]  4.37 GBytes  4.69 Gbits/sec  receiver
[ 5]  Retr 152
```

> 虚拟网卡（vmxnet3）实测 5G+。**别拿"千兆网卡"当上限去解释现象。**

---

## 八、`%ifutil = 557%`：荒谬值

```
12:53:05 PM  ens33  rxkB/s 680120.99  ...  %ifutil 557.16
```

**利用率不可能 >100%**。`sar` 按它读到的链路速率算百分比，虚拟网卡的 `speed` 值不准。

> ★★ 虚拟网卡上 **`%ifutil` 直接忽略**。与 IO 里 `%util` 在 SSD 上不可信同一类：**算法假设在虚拟化下不成立**。
> 正解：用 `rxkB/s × 8` 换算，与已知链路能力比。

---

## 九、重传：为什么 128 是 0、254 报 575

```
128（接收方）：TcpRetransSegs 30 → 30   增量 0
254（发送方）：iperf3 Retr 393 + 182 = 575
```

**两个数都对，统计的是不同机器。**

| 位置 | 计数内容 |
|---|---|
| 发送端 | 自己重传了多少段 → 575 |
| 接收端 | 自己重传了多少段 → 0 |

> ★★ **重传只在发送端计数。** 接收端查 `TcpRetransSegs` 永远看不到对端的重传。

### 重传率看比例

```
4.37 GB / 1448 B ≈ 324 万段，重传 152
152 / 324万 ≈ 0.0047%
```

**<0.1% 属正常**，0.005% 非常健康，>1% 要查链路。

---

## 十、`No route to host` 判据

现场：254 连 128:5201 报 `No route to host`，而 128 上 `ss -lntp` 明明 `LISTEN *:5201`。

| 报错 | 内核 | 真相 |
|---|---|---|
| `Connection refused` | `ECONNREFUSED` | 包到了，**端口没监听** |
| **`No route to host`** | `EHOSTUNREACH` | **被 REJECT**（firewalld ICMP host-prohibited） |
| `timed out` | 无响应 | 包被 **DROP** |

修复：

```bash
sudo firewall-cmd --list-all                 # ports 里没有 5201
sudo firewall-cmd --query-port=5201/tcp      # no
sudo firewall-cmd --add-port=5201/tcp
sudo firewall-cmd --query-port=5201/tcp      # yes → 254 立刻连通
```

> ⚠️ `iptables -S` 查不到东西（firewalld 走 nftables，兼容层空壳），**判放行只用 `firewall-cmd`**。
> ★ 用完必须还：`--remove-port` + `--reload` + `--list-ports` 确认消失。实测最终 `2222/tcp 9100/tcp 10050/tcp`。

---

## 十一、第三次踩同一个坑：观察者错位

```
sar 采样：12:41:11 ~ 12:41:21
iperf3  ：12:50
```

差 9 分钟，`txkB/s` 只有 0.8~14.59，与 5.24 Gbit 完全对不上。

**修正（同块启动）**：

```bash
( sar -n DEV 1 12 > /tmp/net-dev2.log 2>&1 ) &
iperf3 -s -1 > /tmp/iperf-s2.log 2>&1 &
echo ">>> 5 秒内去对端跑客户端 <<<"; sleep 14
grep ens33 /tmp/net-dev2.log | sort -k5 -nr | head -3
```

**结果**：`rxkB/s 680120.99`（峰值），`Average 288320.56`。

> ★★ 这条连踩三次（后台采样器掉队 / sar 早于 dd / sar 早于 iperf）。
> **固定动作：观察者与被观察者写进同一个不可打断的块，先起被观察者，再起负载，最后 sleep 等。**
> 批量粘贴、分步回车，中间那几秒就是误差来源。

---

## 十二、网络判据树

```
应用变慢
  ├─ rxkB/s 接近链路能力        → 带宽打满   → 加链路/分流/限速
  ├─ rxpck/s 高但 rxkB/s 低     → 小包风暴   → 查 DDoS/心跳/DNS 洪水
  ├─ estab 暴涨                 → 连接堆积   → 查连接泄漏
  ├─ timewait 暴涨+端口耗尽     → 短连接多   → 改长连接/连接池
  ├─ TcpRetransSegs 速率持续>0  → 丢包重传   → 查链路/MTU/中间设备
  ├─ TcpOutRsts 高              → 大量 RST   → 端口未监听/被拒
  └─ %ifutil > 100%             → 假数据     → 虚拟网卡忽略
```

---

## 十三、坑位速查（11 条）

| # | 现象 | 正解 |
|---|---|---|
| 1 | `rxkB/s` 当 kbit/s | 千字节，×8÷1024 |
| 2 | `%ifutil` 557% | 虚拟网卡失真，忽略 |
| 3 | `timewait` 多以为故障 | 本机主动关闭，风险是端口耗尽 |
| 4 | `ss \| awk $5` 取空 | 列数随选项变，用 `grep -oE` |
| 5 | `Send-Q 148` 以为发不出 | 是已发未 ACK |
| 6 | `nstat` 累计当当前 | 第 2 列才是速率 |
| 7 | 接收端查重传 | 重传只在发送端计数 |
| 8 | 重传 575 次当链路差 | 看比例 0.005% |
| 9 | `No route` 以为路由坏 | firewalld REJECT |
| 10 | `iptables -S` 判放行 | 只用 `firewall-cmd` |
| 11 | 观察者分块启动 | 同块执行 |

---

## 十四、打脸台账（4 条）

| # | 错误判断 | 实测真相 |
|---|---|---|
| 60 | `iperf3` 可能缺失 | **3.18 已装**；缺的是 `nc` |
| 61 | `TcpRetransSegs` 应为 0 | **30（累计）**，速率 0.0 |
| 62 | `ss` 的 `$5` 是对端 | **空值**，列数随选项变 |
| 63 | 带宽 1.5~2.5 Gbit/s | **5.24 Gbit/s** |

**共同点**：60/61/63 是"按经验估数"，62 是"按上次的列位置套这次的命令"。
**唯三对的（`No route`=未放行、重传在发送端、`sar` 单位是 kB）全部来自判据树，不是记忆。**

---

## 十五、一句话总结

> 「网络账分四小账。带宽用 `sar -n DEV`，`rxkB/s` 是千字节，乘 8 除 1024 才是 Mbit——我实测 680120 kB/s 对上 iperf3 的 5.24 Gbit/s，两条命令互证。连接用 `ss -s`：`estab` 是实时并发，`timewait` 多只说明本机主动关了很多连接、真风险是端口耗尽不是 TIME_WAIT；本地端口是连续高位随机端口就是客户端。丢包用 `nstat`，第 1 列开机累计、第 2 列才是当前速率，而且**重传只在发送端计数**——接收方 `TcpRetransSegs` 增量 0、对端报 152 次，两个数都对，看比例 0.005% 属健康。两个坑：`%ifutil` 在虚拟网卡上算出 557% 这种荒谬值要忽略；`No route to host` 不是路由坏，是 firewalld REJECT，`firewall-cmd --query-port` 一查就露，用完记得还回去。」
