# DAY6 · 网络（2026-09-19 上午 · 128 / oe-base）

> 今天主题：**认门牌 → 改门牌 → 守大门**
> 环境：网卡 `ens33`（VMware，MAC 00:0c:29:39:52:e2），IP 192.168.252.128/24，网关 192.168.252.2

---

## 一、认门牌：四条体检命令

```bash
nmcli device status          # 设备视角
nmcli connection show        # 连接（配置文件）清单，NAME 才是改配置的名字
ip -br a                     # 内核视角（brief 精简）
ip r                         # 路由表
```

| 现象 | 含义 |
|---|---|
| `lo  connected (externally)` | **externally = 内核自己管，NetworkManager 不插手** |
| `lo  UNKNOWN`（ip -br a） | ⚠️ **不是故障** —— 环回没有载波概念，内核就报 UNKNOWN |
| `proto static`（default 路由） | 配置里写死的 |
| `proto kernel`（同网段路由） | 内核按 IP + 掩码**自动生成** |
| `scope link` | 直连可达，不过网关 |
| `metric 100` | 路由优先级，**数字小者优先** |

**彩蛋 · EUI-64 反推 MAC**：`fe80::20c:29ff:fe39:52e2` → 剥掉中间 `ff:fe` → **00:0c:29:39:52:e2**（`00:0C:29` = VMware OUI）。
规则：MAC 中间劈开、插 `ff:fe`、第一字节第 7 位置 1。`ip -br link` 可验证。

类比：**`nmcli` = 人事档案（配置层），`ip` = 今天谁真来上班了（内核层）** —— 档案改了不等于人来上班。

---

## 二、配置落盘的"双面孔"

```bash
sudo cat /etc/NetworkManager/system-connections/ens33.nmconnection   # → 文件不存在
sudo ls -l /etc/NetworkManager/system-connections/                   # → total 0（空目录）
sudo ls -l /etc/sysconfig/network-scripts/                           # → ifcfg-ens33 ✅
sudo cat /etc/sysconfig/network-scripts/ifcfg-ens33
```

**真相**：openEuler 的 NetworkManager 用 **ifcfg-rh 插件**（读传统 RHEL 格式），不是 keyfile（`.nmconnection`，权限 600）。

**双语对照**

| ifcfg 老写法 | nmcli 新字段 |
|---|---|
| `BOOTPROTO=static` | `ipv4.method: manual` |
| `ONBOOT=yes` | `connection.autoconnect: yes` |
| `IPADDR` + `PREFIX` + `GATEWAY` | `ipv4.addresses` + `ipv4.gateway` |
| `DNS1` / `DNS2` | `ipv4.dns` |
| `DEFROUTE=yes` | `GENERAL.DEFAULT: yes` |
| **`IPV4_FAILURE_FATAL=no`** | **`ipv4.may-fail: yes`（逻辑相反）** |
| `IPV6_ADDR_GEN_MODE=eui64` | `ipv6.addr-gen-mode: eui64` |

**铁证**：ifcfg 的 `UUID` 与 `nmcli` 的 `connection.uuid` **完全一致** → 同一份配置的两个面孔。

**坑**
- ❌ `systemctl restart network` 在 openEuler 24.03 **已失效**
- ✅ 手改 ifcfg 后：`nmcli con reload` + `nmcli con up ens33`；**只 reload 不 up = 白改**

---

## 三、多 IP 热生效 + 正反闭环

```bash
sudo nmcli connection modify ens33 +ipv4.addresses 192.168.252.188/24   # 追加（无输出=成功）
sudo nmcli device reapply ens33                                          # 热生效，SSH 不断
ip -br a show ens33                                                      # 两个 IPv4 并列
ping -c 2 192.168.252.188            # 本机 0.052ms
# 254 上：ping -c 2 192.168.252.188  # 0.474ms（穿了一次虚拟交换，延迟差 ≈10 倍）

sudo grep -E 'IPADDR|PREFIX' /etc/sysconfig/network-scripts/ifcfg-ens33
# IPADDR=192.168.252.128 / PREFIX=24 / IPADDR1=192.168.252.188 / PREFIX1=24  ← 多 IP 用数字后缀

sudo nmcli connection modify ens33 -ipv4.addresses 192.168.252.188/24   # 删除
sudo nmcli device reapply ens33 && ip -br a show ens33                   # 只剩 .128
# 254 上 ping → 100% packet loss（通→不通 = 因果证据闭环）
```

**坑**
- `+` 追加 / `-` 删除 / **不加符号 = 覆盖（危险）**
- `reapply` = **行驶中换轮胎**（不断线）；`con up` = 靠边停车重新出发
- 只看本机 ping 就下结论 ❌ → 必须从对端交叉验证

**彩蛋**：254 自己有三个 IP —— 192.168.252.254 + **172.17.0.1（docker0）** + **172.19.0.1（kind 网桥）**。

---

## 四、守大门：firewalld 区域与两本账

```bash
systemctl status firewalld --no-pager    # enabled + active；🎁 firewalld 是 Python 进程
sudo firewall-cmd --state                # running
sudo firewall-cmd --get-active-zones     # public + interfaces: ens33
sudo firewall-cmd --list-all             # 运行时规则
```

```
public (active)
  services: dhcpv6-client http mdns ssh     ← ssh 在册，会话安全
  ports: 2222/tcp                            ← 9/14 改 SSH 端口留下的痕迹
```

- **区域（zone）= 信任级别**：`trusted` > `home`/`work`/`internal` > `public`/`external` > `dmz` > `block` > `drop`
- 两种放行方式：**service**（预定义模板，如 ssh/http）vs **裸端口**（如 2222/tcp）
- `connection.zone: --` → 落默认 `public` 区

**两本账**

| 命令 | 看的是 |
|---|---|
| `firewall-cmd --list-all` | **运行时**（内存） |
| `firewall-cmd --list-all --permanent` | **永久**（磁盘，重启/reload 后加载） |

> 永久版 `interfaces:` 空是**正常**的 —— "接口绑哪个区"是运行时概念。

**考古**：`/etc/firewalld/zones/public.xml`（415B，9/14 09:30）+ `public.xml.old`（378B，9/11 10:24）
- `.old` = firewalld **自动备份** → 免费的配置变更历史
- 出厂模板只有 ssh/mdns/dhcpv6-client；`diff` 输出 `8a9 > <port port="2222" protocol="tcp"/>` → 还原改动史

**坑**
- `sudo stat /dir/*` 报 No such file → **`*` 由当前 shell 展开，sudo 不提权展开**；用 `sudo sh -c "stat … *"`
- 改 firewalld 前先看清单，别把自己关在门外

---

## 五、收官大戏：「服务在、端口不通」

```bash
# ① 128 起监听
cd /tmp && nohup python3 -m http.server 8080 --bind 0.0.0.0 >/tmp/http8080.log 2>&1 &
sudo ss -tlnp | grep 8080
# LISTEN 0 5 0.0.0.0:8080 0.0.0.0:* users:(("python3",pid=7173,fd=3))
curl -m 3 http://127.0.0.1:8080/ | head -5      # 本机通 → 服务自证清白

# ② 254 打过来
curl -m 5 -v http://192.168.252.128:8080/
# → No route to host，1 ms
```

**★★★★ 四症状 × 四成因（排障听诊器）**

| 报错 | 成因 | 特征 |
|---|---|---|
| `Connection refused` | ①没人监听（RST）②防火墙 `REJECT … icmp-port-unreachable` | **两种成因报错一样**，用 ss 排除① |
| `No route to host` | 防火墙 `REJECT … icmp-host-prohibited`（firewalld public 默认） | **毫秒级失败** |
| `Connection timed out` | 防火墙 `DROP`（ICMP 都不回） | 最难查，沉默 |
| 正常 | — | — |

> **"本机能通"≠"外网能通"**：环回在 INPUT 链被 `-i lo -j ACCEPT` 提前放行。

**③ 波形（不通 → 通 → 蒸发 → 不通 → 通 → 清理）**

```bash
sudo firewall-cmd --add-port=8080/tcp                      # success；仅运行时 → 254 立刻 200 OK
sudo firewall-cmd --list-all --permanent | grep -A1 ports   # 永久账本没有 8080（伏笔）
sudo firewall-cmd --reload && sudo firewall-cmd --list-all | grep -A1 ports
# 8080 蒸发 → 254 又 No route to host（0 ms）
sudo firewall-cmd --permanent --add-port=8080/tcp && sudo firewall-cmd --reload
# 两本账都有 8080 → 254 复测 200 OK ✅
sudo firewall-cmd --permanent --remove-port=8080/tcp && sudo firewall-cmd --reload   # 减法对称
pkill -f "http.server 8080"                                # Terminated; [1]+ Exit 143
```

**🎁 三个彩蛋**
1. `Exit 143` = 128 + **15 = SIGTERM**（pkill 默认礼貌请退）；强杀 `-9` → 137。与 OOM 的 137 同一套算法
2. `/tmp` 列表里的 `systemd-private-…-chronyd/nginx/polkit/logind…` = systemd **`PrivateTmp=yes`** 沙箱隔离
3. HTTP 头 `Date: … 03:10:30 GMT`（本地 11:10 CST）→ 协议规定用 GMT，**"同一事件两个时区视角"**；`Server: SimpleHTTP/0.6 Python/3.11.6` 暴露版本号 → 生产要 `server_tokens off`

**坑**
- `--add-port` 不带 `--permanent` = 临时通行证，reload/重启即失
- 顺序：**先 `--permanent` 后 `reload`**
- 收工判据：**两本账一致**，不是"现在能通"
- 实验后对称清理，别留脏规则
- 作业号 ≠ PID（`[1] 7172` vs `pid=7173`）

---

## 六、今日坑清单（汇总）

| # | 坑 | 一句话 |
|---|---|---|
| 1 | lo 报 UNKNOWN | 不是故障，环回无载波概念 |
| 2 | ens33 连接名混用 | `nmcli con mod` 认连接名（本例恰好同名） |
| 3 | 配置找不到 | NM 走 ifcfg-rh，配置在 `/etc/sysconfig/network-scripts/` |
| 4 | `systemctl restart network` 已失效 | 用 `nmcli con reload` + `con up` |
| 5 | `+`/`-`/`=` 三兄弟 | 不加符号 = 覆盖 |
| 6 | reapply 被忽略 | reapply = 不断线热生效，改完必须复核内核层 |
| 7 | 只本机 ping 就下结论 | 必须对端交叉验证 |
| 8 | `sudo cmd *` | 通配符由当前 shell 展开，用 `sudo sh -c` |
| 9 | 只看运行时账本 | 两本账不一致 = 早晚出事 |
| 10 | 先 reload 后 permanent | 顺序反了规则被清 |
| 11 | `No route to host` 误判成网络故障 | 是 REJECT（有 ICMP 回执、毫秒级） |
| 12 | 实验不留清理 | 加减对称 |

---

## 七、面试话术

**① 网卡配置**
> "改网卡我用 `nmcli`：`+/-ipv4.addresses` 增删地址、`nmcli device reapply` 热生效不断线；改完 `ip -br a` 复核内核层、从对端交叉验证。openEuler 的 NM 走 ifcfg-rh 插件，配置落在 `/etc/sysconfig/network-scripts/ifcfg-<iface>`，与 `nmcli` 的 connection.uuid 一致；手改文件后要 `con reload` + `con up`，`systemctl restart network` 已经是老黄历。"

**② 防火墙**
> "firewalld 我记'区域 = 信任级别'和'两本账'：`--list-all` 看运行时、加 `--permanent` 看永久。放行必须 `--permanent` + `reload` 两步，只加运行时 reload 就没 —— 这是'改完是好的、重启又不行'的根源。另外它有个 `.old` 自动备份，能还原改动史。"

**③ 连通性排障（最值钱）**
> "客户端连不上我按三层查：①`ss -tlnp` 看进程是否监听 —— 注意**本机 curl 通不代表外网通**，环回被 `-i lo` 提前放行；②`firewall-cmd --list-all` 核对运行时与永久;③再看 SELinux。
> **报错形态直接定成因**：`Connection refused` = 没人监听或 port-unreachable（两种成因同形，用 ss 排除）；`No route to host` = host-prohibited，firewalld public 区默认 REJECT；`Connection timed out` = DROP，最阴的一种。"
