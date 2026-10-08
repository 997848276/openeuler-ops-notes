# day28 · 网络排障判据树（2026-10-07 · 128 oe-base · openEuler 24.03）

## 一 防火墙后端对质：iptables 空壳 vs nft 真账本

```bash
ip -br addr show | grep -v lo            # ens33 UP 192.168.252.128/24
ip route show default                    # default via 192.168.252.2 dev ens33 metric 100
cat /etc/resolv.conf | grep -v '^#'      # 114.114.114.114 + 8.8.8.8

sudo firewall-cmd --state                # running
sudo firewall-cmd --get-default-zone     # public
sudo iptables -S | wc -l                 # 3   ← 只有 policy，零规则（空壳！）
sudo nft list ruleset | wc -l            # 337 ← table inet firewalld 真账本
sudo nft list ruleset | grep -i reject | head -5
# reject with icmpx admin-prohibited     ← 默认 REJECT 的 nft 真身（v4/v6 一条通吃）
```

- openEuler 24.03 firewalld = **nftables 原生后端**；`iptables -L` 看到"空"≠没规则。
- 对比：CentOS 7 = iptables 后端；迁移时查/备份防火墙命令要换（iptables-save → nft list ruleset / firewall-cmd）。

## 二 监听面侦察：ss 三件套

```bash
sudo ss -tulpn | head -15                # 监听口 + 进程/pid/fd
sudo ss -s                               # TCP 13 (estab 2) / UDP 6
sudo ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn    # 11 LISTEN / 2 ESTAB
sudo ss -tulpn | grep -E ':(22|2222|80|111)\b'
```

- sshd(1225) 一个进程监听 22+2222 × v4+v6 = **4 个 socket**（fd 3/4/5/6）。
- LISTEN 统计是双栈重复计数；2 个 ESTAB = 自己的 SSH。
- 127.0.0.1:3838 / 127.0.0.1:10350 只听回环 = 端口扫描扫不到——**绑定地址本身就是一道墙**。

## 三 同一个端口三种死法：firewalld 状态切换实验

```bash
# 128 起靶（注意 --directory 隔离，别绑家目录）
sudo sh -c 'nohup python3 -m http.server 9999 --bind 0.0.0.0 --directory /tmp >/tmp/http9999.log 2>&1 &'
sudo firewall-cmd --list-all             # services: http mdns ssh... / ports: 2222/tcp 9100/tcp

# 254 跨机打靶（每发之间回 128 改状态）
curl -m 5 http://192.168.252.128:9999/ ; echo "rc=$?"    # 未放行 → rc=7 秒拒
sudo firewall-cmd --add-rich-rule='rule family=ipv4 source address=192.168.252.254 port port=9999 protocol=tcp drop'
curl -m 5 http://192.168.252.128:9999/ ; echo "rc=$?"    # DROP → rc=28 timeout
sudo firewall-cmd --remove-rich-rule='...'; sudo firewall-cmd --add-port=9999/tcp
curl -m 5 http://192.168.252.128:9999/ ; echo "rc=$?"    # 放行 → rc=0
```

- **时序判别**：rc=7（1ms）= 对面有人回话；rc=28（5s 等满）= 石沉大海。
- ⚠️ 本机 curl 自己的 IP 未放行却通 → **回环走 lo，firewalld 放行 lo；防火墙实验必须跨机打**。
- ⚠️ `sudo cmd &` 后台 = T 态；正解 `sudo sh -c 'nohup … &'`。

## 四 定罪之路：curl 文案会撒谎，errno 不会

- 9100（放行+没人听）与 17777（未放行），curl 8.x 报错**一字不差**（文案已合并）。
- bash /dev/tcp 报错直接 strerror(errno)，内核原话：

```bash
timeout 5 bash -c 'echo > /dev/tcp/192.168.252.128/9100'  ; echo "rc=$?"
# bash: connect: Connection refused        ← ECONNREFUSED = 内核 RST（放行+没听）
timeout 5 bash -c 'echo > /dev/tcp/192.168.252.128/17777' ; echo "rc=$?"
# bash: connect: No route to host          ← EHOSTUNREACH = ICMP 拒绝信（未放行）
timeout 5 bash -c 'echo > /dev/tcp/192.168.252.128/2222'  ; echo "rc=$?"
# 静默 rc=0                                 ← 三次握手成功
```

## 五 判据树终稿

| 现象 | errno | 真相 | 修法 |
|---|---|---|---|
| 通 | 0 | 三次握手 | — |
| Connection refused | ECONNREFUSED | RST：放行+没人坐 | 起服务 |
| No route to host | EHOSTUNREACH | ICMP admin-prohibited：未放行 | firewall-cmd --add-port |
| timeout | — | SYN 重传无回应 | 查 DROP 规则/路由 |

- 排障顺序：`ss -tlnp`（听没听）→ `--list-all`（放没放）→ 报错对号 → `nft list ruleset` 看原文。

## 六 坑位速查

| 坑 | 正解 |
|---|---|
| iptables -L"空" | nftables 后端，用 nft list ruleset |
| 本机自测防火墙"永远通" | 走 lo 不过 zone，跨机打 |
| curl 合并文案 | bash /dev/tcp 看 errno |
| sudo cmd & 卡死 | sudo sh -c 'nohup … &' |
| http.server 暴露家目录 | --directory /tmp |
| runtime 规则重启丢 | --permanent + --reload（实验用 runtime 正好即清场） |
| nohup 文件是报错 | 落盘≠成功，读文件才知道 |

## 面试一句话

> "端口不通我先分四种死法：refused=放行但没人听（内核 RST）、No route to host=firewalld 没放行（ICMP admin-prohibited）、timeout=DROP（SYN 重传到死）——curl 新版把前两种文案合并了，我用 bash /dev/tcp 让 connect 的 errno 说原话。服务端先 ss -tlnp 验监听、firewall-cmd --list-all 验放行，查规则直接 nft list ruleset——因为 openEuler 24.03 的 firewalld 走 nftables 后端，iptables -L 只看得到空壳，这点和 CentOS 7 完全不同。"
