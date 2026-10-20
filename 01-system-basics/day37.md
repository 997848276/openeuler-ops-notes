# day37 · firewalld 进阶：富规则、端口转发与 refused 双凶手案（2026-10-19 · 128 oe-base）

## 一 活账本：先把当前规则看全

```bash
sudo firewall-cmd --list-all
```

实测（128 public zone）：`services: http`、`ports: 2222/tcp 9100/tcp`、rich rules 为空。

- `--list-all` = **活账本**（内存里生效的），`--list-all --permanent` 才是**存档本**（写盘的）。
- 两条命令结果不一致 = 你改了规则没 reload，或者改了没加 --permanent。

🎯 类比：活账本=**柜台上的实时价目表**，存档本=**备案的档案**。改价目表要两份都对上，明天开店才不会打架。

## 二★ 富规则：按「谁从哪来、找谁、干什么」点名

```bash
# 拒绝 254 访问本机的 22 端口，并明确回音
sudo firewall-cmd --add-rich-rule='rule family="ipv4" source address="192.168.252.254" port port="22" protocol="tcp" reject'
# 改成装死
sudo firewall-cmd --remove-rich-rule='rule family="ipv4" source address="192.168.252.254" port port="22" protocol="tcp" reject'
sudo firewall-cmd --add-rich-rule='rule family="ipv4" source address="192.168.252.254" port port="22" protocol="tcp" drop'
```

跨机实测（在 254 上打 128）：

| 动作 | 254 侧感受 | 本质 |
|---|---|---|
| `reject` | `Connection refused`（**秒回**） | 回 RST / ICMP unreachable，**有回音** |
| `drop` | `Connection timed out`（**卡到超时**） | 丢包不回话，**装死** |

🎯 类比：reject=门卫探出头说"不让进"（你立刻知道被拒）；drop=**门卫假装没听见**（你在门口等到地老天荒）。

## 三★★ refused 双凶手案（修正 §三十四 判据树）

以前写「refused = 端口后面没进程」，今天实测补上一刀：

| 应用层感受 | 可能的两个凶手 | 怎么区分 |
|---|---|---|
| `Connection refused` | ①**端口后没进程**（内核 12µs 代答 RST，应用层没参与）<br>②**防火墙 reject**（回 RST/ICMP unreachable，connect 同样报 ECONNREFUSED） | ★**先查 `firewall-cmd --list-all` 看有没有 reject 富规则，再查 `ss -lntp` 看端口有没有人听** |
| `Connection timed out` | **只有一种**：DROP / 包被丢 | 抓包看 SYN 重传（§四十三：重传不改 seq） |
| `No route to host` | ICMP unreachable（网关/路由层） | 走的是 ICMP，不是 TCP |

📌 判据树修正：**refused 从"一个凶手"变"两个嫌犯"**，timeout 仍是一凶手。别再一见到 refused 就冲去看服务。

## 四 端口转发：把 8081 的活转给 80

```bash
sudo firewall-cmd --add-forward-port=port=8081:proto=tcp:toport=80
sudo firewall-cmd --add-port=8081/tcp                  # 别忘了开新门的放行
# 254 上验证
curl -s http://192.168.252.128:8081 | head -1
```

实测回显：`<h1>oe128 - deployed by Ansible</h1>` ✅

- 本质 = **DNAT**（改目的端口），和 Docker `-p` 是同一套内核机制（§十八 里 nat 表 DOCKER 链）。
- 区别在于：Docker 那条链**只认网卡不是网桥的流量**（`! -i docker0`），firewalld 这条**认所有进来的**。

配套概念（本次仅入册未实操）：`masquerade` = **出方向的源地址伪装**（内网机器借防火墙的外网 IP 出门），端口转发管"进来"，masquerade 管"出去"。

## 五 坑位速查表

| # | 现象 | 正解 |
|---|---|---|
| 1 | 加了规则不生效 | ★**`--permanent` 写盘 + `--reload` 进内核，双轨缺一不可**；只加 permanent 不 reload = 写了没生效；只 reload 不加 permanent = 重启即蒸发 |
| 2 | remove 报 NOT_ENABLED | 删规则要把**原字符串原样写回**，改一个空格都认不出来 |
| 3 | 判退出码时被管道骗 | `cmd \| head -2; echo $?` 拿到的是 **head 的 rc**（实测拿到 0，差点误判成功）；正解 **`${PIPESTATUS[0]}`** |
| 4 | 见 refused 就查服务 | 先查防火墙（本案新增凶手） |
| 5 | 转发配了不通 | 新端口自己也要放行（转发 ≠ 放行） |

## 六 面试一句话

> 「firewalld 的富规则能按源 IP + 端口精确点名，reject 会回音（客户端秒见 refused），drop 装死（客户端等到 timeout）——所以**排障时见到 refused 不能只查服务，还得先查防火墙是不是在 reject**；
> 端口转发本质是 DNAT，和 Docker 的 -p 是同一套内核机制，区别在于 Docker 那条链只管非网桥流量；
> 改规则要记住两条轨道：`--permanent` 是写盘、`--reload` 是进内核，缺一条就是'改了没生效'或'重启就丢了'。」
