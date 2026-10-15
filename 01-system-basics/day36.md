# day36 · tcpdump 抓包：把排障口诀升级成一手证据（2026-10-15 · 128 oe-base）

## 一 装备与语法四件套

```bash
rpm -q tcpdump || sudo dnf install -y tcpdump    # 128 经 OS 仓装（473k）
sudo tcpdump -D                                   # ens33 / any / lo [Loopback]
sudo tcpdump -i ens33 -nn -c 8 icmp
```

| 参数 | 含义 | 坑位 |
|---|---|---|
| `-i ens33` | 蹲指定网卡 | 回环流量只在 `-i lo` |
| `-nn` | 不解析，IP/端口全显示数字 | 又快又不糊 |
| `-c N` | 抓满 N 包自动停 | ⚠️**包不够不收工**——配 wait 会挂死，永远套 `timeout 10` 前缀（实测 -c 8 挂 2 分钟等别人家的包凑数） |
| 过滤表达式 | `icmp` / `tcp port 80` / `host x.x.x.x` | 写在命令尾 |

📌 tcpdump 的天性：**蹲在路口连别人家的事故都录**——实测录到 254 机自发发的 `ICMP host unreachable - admin prohibited filter`（353B 大包）。

## 二 三次握手眼见为实（curl 本机 nginx，抓 lo）

```
IP 127.0.0.1.20922 > 127.0.0.1.80:  Flags [S],  options [mss 65495,sackOK,TS val.. ecr 0,nop,wscale 7]
IP 127.0.0.1.80  > 127.0.0.1.20922: Flags [S.]
IP 127.0.0.1.20922 > 127.0.0.1.80:  Flags [.]
IP 127.0.0.1.20922 > 127.0.0.1.80:  Flags [P.], length 72:  HTTP: GET / HTTP/1.1
IP 127.0.0.1.80  > 127.0.0.1.20922: Flags [P.], length 273: HTTP: HTTP/1.1 200 OK
```

- Flags 破译：**S=SYN、.=ACK、P=PSH（带数据）、F=FIN、R=RST**（组合写 [S.]、[P.]）。
- options 里 mss/sackOK/TS val/ecr/wscale = **双方见面互递名片**（最大段长、SACK、时间戳、窗口缩放）——以后抓慢 TCP 的线索全在这。
- 回环走 lo **不过 firewalld 的 zone**（§三十四），但 netfilter 的 INPUT hook 照走。

## 三★ 判据树三态包级定罪表（§三十四 收官）

| 应用层感受 | 包层面实测 | 谁在说话 |
|---|---|---|
| `Connection refused` | SYN → **12 微秒后 [R.]**（win 0） | **内核代答**——端口后没进程，应用层没参与拒绝 |
| 卡死→timeout（curl exit=28） | **同一 seq 的 [S] 发 4 遍**（~1s 重试），零回包 | **DROP 装死**；★**重传不改 seq=连接从没建立** |
| `No route to host` | ICMP unreachable（type 3） | 路由器/网关拦路 |

**DROP 实验**（iptables 直写 hook，绕过 firewalld 的 zone 绑定）：

```bash
sudo iptables -I INPUT -i lo -p tcp --dport 9998 -j DROP   # 设黑洞
# tcpdump 只见 [S] ×4 同 seq 重发，无回包；curl exit=28
sudo iptables -D INPUT -i lo -p tcp --dport 9998 -j DROP   # 还原（-S INPUT 只剩 -P ACCEPT）
```

📌 方法论：**"refused/timeout" 是应用层的转述，RST/静默才是包层面的一手证据**——排障升维就是"从听转述到看现场"。

## 四 坑位速查

| 坑 | 正解 |
|---|---|
| tcpdump -c N 配 wait 卡死 | 包不够不收工；**永远 `timeout 10 tcpdump ...`** |
| 满屏杂音 | -c 限量 + 精确过滤表达式 |
| 抓回环没流量 | 回环在 `-i lo`，不在 ens33 |
| firewalld 对回环不生效 | lo 不在 zone；要管 lo 用 iptables 直写 hook |
| 读图定罪时间戳 | ⚠️本次打脸：07:59 看成 06:59 虚报"时钟慢 1h"，实机 date/chronyc 无罪释放——**关键值先实机验证再定罪** |
| 抓包文件撑爆盘 | 长期用 `-w file.pcap` + `-C` 切割；临时排查抓完即停 |

## 面试一句话

> 「网络排障我分两层看：应用层的报错是转述，包层面才是一手证据——connection refused 在抓包里是内核 12 微秒代答的 RST，说明端口后根本没进程；
> 超时是同一个 seq 的 SYN 反复重传石沉大海，说明防火墙在 DROP；
> 抓包三件套：-i 指定接口、-nn 不解析、-c 限量加 timeout 防挂死；
> 另外回环流量走 lo 不过 firewalld 的 zone，但 netfilter 的 hook 照走，所以 iptables 直写规则就能管住回环。」
