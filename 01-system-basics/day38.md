# day38 · ss 与连接状态排查：socket 表的视野边界（2026-10-20 · 128 oe-base）

## 一 全家桶侦察与账本勾稽

```bash
sudo ss -lntup     # 监听表：-l 监听 -t tcp -u udp -p 进程（必须 sudo）
sudo ss -s         # 总账
sudo ss -tan | awk 'NR>1{print $1}' | sort | uniq -c | sort -rn   # 状态普查
```

实测：普查 `12 LISTEN + 3 ESTAB`，`ss -s` 报 TCP 15——**两本账勾稽分毫不差**。

- 户口盘点：sshd **22+2222**｜nginx **80+8008**｜rpcbind 111｜node-exporter 9100｜isulad 10350｜**127.0.0.1:3838 = atuned 的 python 分析引擎**（app_engine.py）。
- ⭐ 活体发现：`ESTAB 128:835 ← 254:2049`——**NFS 挂载是一条常驻 TCP 连接**。

📌 ss=电话局通话记录簿：只登记「接通了的」和「正在等的」。

## 二★ Recv-Q / Send-Q 两副面孔（本章核心）

| 状态 | Recv-Q | Send-Q | 类比 |
|---|---|---|---|
| LISTEN | 0 | **应用声明的 backlog**（nginx 511 / sshd 128 / rpcbind 4096 / 9100=8192） | 店门口候座位子数 |
| ESTAB | 收了没读的字节 | 发了没被确认的字节 | 柜台上没取/没送的包裹 |

- 天花板 `net.core.somaxconn = 8192`（本机实测）——LISTEN 的 Send-Q = min(应用 backlog, somaxconn)；9100 的 8192 是**顶格合法**。
- `tcp_max_syn_backlog = 256` 是另一个队列（半连接），别混。

**喂队列实验**（服务端 accept 后装死，客户端灌 2MB）：

```
ESTAB  0        1970001   127.0.0.1:57638 → 127.0.0.1:19999   ← 客户端
ESTAB  127151   0         127.0.0.1:19999 → 127.0.0.1:57638   ← 服务端
```

**字节守恒**：127151 + 1970001 = 2097152，分毫不差。大头堵在**发送方**——内核接收缓冲默认 ~128KB 是第一道天花板：对方不收货，快递员的车斗（Send-Q）先爆，不是仓库（Recv-Q）。

生产判据：**LISTEN 的 Recv-Q 长期 > 0 = 应用 accept 太慢**；`nstat` 的 **ListenOverflows 涨 = 全连接队列打满**。

## 三 TIME_WAIT：谁守灵、堆多了炸什么

```bash
for i in $(seq 1 200); do curl -s -o /dev/null --max-time 2 http://127.0.0.1:80/; done
sudo ss -tan state time-wait | wc -l        # → 201，全在客户端侧
sudo nstat -az | grep -iE 'ListenOverflow|ListenDrops'   # → 0
```

- **TIME_WAIT 归主动关闭方**（谁先 FIN 谁守灵）：200 条短连接全堆在 curl 侧，nginx 干干净净；约 5 分钟后衰减回 1。
- ⚠️ **两个"60"不是一回事**：`tcp_fin_timeout=60` 管 FIN_WAIT_2；**TIME_WAIT 的 60 秒内核写死（TCP_TIMEWAIT_LEN），sysctl 改不了**——面试混说直接露馅。
- 端口容量账：`ip_local_port_range = 10240 65535`（55296 个灵位 ÷ 60s ≈ **921 条短连接/秒打满**），打满症状 = `Cannot assign requested address`。
- `tcp_tw_reuse = 2` = **仅回环可复用**；跨机出站要开 1。正解优先级：**长连接 > 连接池 > tw_reuse**。

## 四★★ socket 表的视野边界：三种"看不见"

OUTPUT DROP 掉 :80 出站 SYN 的实验，三轮才跑干净，每轮钉死一个坑：

| # | 坑 | 破案 |
|---|---|---|
| 1 | 前置条件（iptables -I）没进命令组 | 实验压根没做——**前置条件必须写进命令组**（老教训重犯） |
| 2 | `ss -tn` 找不到刚断的连接 | **TIME_WAIT 不带 -a 不显示**，查连接一律 `ss -tan` |
| 3 | `grep ':80 '` 匹配不到 SYN-SENT | **尾空格只匹配 Local 列**（行尾 Peer 端口隐身）→ 正解 = ss 原生过滤 `'( sport = :80 or dport = :80 )'` |
| 4 | `wait; echo $?` 报 0，实际 Exit 124 | **wait 吞 rc** → 子 shell 记账：`( cmd; echo "RC=$?" ) &` |
| 5 | tcpdump 8 秒窗口被人肉粘贴耗光 | **并发实验写成单条命令链一次粘贴** |
| 6 | DROP 生效但 tcpdump **0 packets** | **★OUTPUT 链丢的包从未上线**——tcpdump 蹲在设备层只看得见上了线的包；INPUT 丢的已上线（抓得到），OUTPUT 丢的是胎死腹中 |

SYN-SENT 现行：`SYN-SENT 0 1 127.0.0.1:10814 → 127.0.0.1:80`——**正在反复敲门，永远敲不开**。

三层证据定罪：**应用层（curl rc）→ socket 表（ss）→ 包层面（tcpdump）**；OUTPUT 丢弃的连接 ss/tcpdump 双双无 record，唯一证据是应用层 rc。

## 五 面试一句话

> 「ss 看连接记住 Recv-Q/Send-Q 两副面孔：监听态是队列长度（Send-Q=应用 backlog、somaxconn 封顶），建立态是字节数（Recv-Q 堆着=应用读不动，Send-Q 堆着=对端收不动）；
> TIME_WAIT 是主动关闭方的 60 秒守灵，短连接打满耗尽本地端口报 Cannot assign requested address，正解长连接、连接池、出站开 tw_reuse，别碰废弃的 tw_recycle；
> 排障上 ss 只看得见 socket 表里的连接——正常断开的在 TIME_WAIT 要加 -a，被 OUTPUT 链丢弃的连接连包都没上线、tcpdump 都抓不到——定罪要三层证据齐上：应用层 rc、ss 状态机、tcpdump 包层面，SYN-SENT 卡着不动就是'正在敲门永远敲不开'。」
