# openEuler 运维速查表 · 第 1 周

> 覆盖：2026-09-10 ~ 09-19（DAY1~DAY6）
> 环境：openEuler 24.03 LTS（练习机 128，SELinux Enforcing）
> 用途：面试前 10 分钟速扫 / 日常排障手边卡

---

## 一、排障判据速查（★ 最值钱，先看这节）

**客户端连不上 → 按四层漏斗下钻：**

| 层 | 怎么查 | 正常表现 |
|---|---|---|
| ① 链路可达 | `ping <IP>` | 0% loss |
| ② 进程监听 | `sudo ss -tlnp \| grep <端口>` | LISTEN + 0.0.0.0 + pid |
| ③ 防火墙放行 | `sudo firewall-cmd --list-all` | 端口/服务在册 |
| ④ SELinux 允许 | `sudo semanage port -l \| grep <端口>` | 端口类型与进程域匹配 |

**症状 → 成因（背这张表，比背命令有用）：**

| 客户端症状 | 成因 | 判断关键 |
|---|---|---|
| `Connection refused` | ① 没人监听（内核回 RST）② 防火墙 `REJECT --reject-with icmp-port-unreachable` | **两种成因同形**，必须用 `ss -tlnp` 排除① |
| **`No route to host`** | 防火墙 `REJECT --reject-with icmp-host-prohibited`（**firewalld public 区默认**） | **毫秒级失败**（有 ICMP 回执） |
| `Connection timed out` | 防火墙 `DROP`（连 ICMP 都不回） | **最难查**：沉默无反馈 |
| 本机能通、外网不通 | 防火墙/监听层 | 环回被 `-i lo -j ACCEPT` 提前放行 → **本机通≠外网通** |
| 配置语法对但起不来 | **SELinux** | 典型特征：`nginx -t` 通过、服务起不来 |
| `bind() failed (13: Permission denied)` | SELinux 端口白名单 | 动作是 `bind()` → 跟文件权限无关 |

**退出码速查（减 128 = 凶手信号号）：**

| 码 | 含义 |
|---|---|
| `137` | 128+9 = **SIGKILL**（不可捕获；OOM 被杀典型） |
| `143` | 128+15 = **SIGTERM**（可捕获；`pkill` 默认发的就是它） |
| `1` | 程序自己报错退出 |
| `203` | systemd 执行 ExecStart 失败（查 SELinux 标签/权限） |
| `209` | systemd 写不了 stdout 日志（路径不可写） |

**systemd 判决书**：`systemctl status` 里的 `status=` → `2xx` = systemd 没铺好摊子；`1` = 脚本自己挂。

---

## 二、包管理（rpm / dnf）

```bash
rpm -qi <包>          # 包信息        rpm -ql <包>      # 装了哪些文件
rpm -qf <文件>        # 文件属于哪个包  rpm -qc <包>     # 配置文件在哪
rpm -Va              # 全量校验（对比原始信息，查篡改/丢失）
rpm -qlp xx.rpm      # 看未安装包的内容
dnf provides <文件>   # 谁知道这个命令/文件属于哪个包
dnf download <包>     # 离线下载不安装
dnf history          # 事务审计；dnf history undo <ID>
```

---

## 三、用户与权限

```bash
useradd -m -G appgrp devops     # 建用户 + 家目录 + 附加组
usermod -aG wheel devops        # 追加组（-a 必须带，否则覆盖）
id devops                        # 看 uid/gid/组

chmod 4755 / 2775 / 1777         # SUID / SGID / Sticky
ls -l                             # s=SUID/SGID 生效  S=空设  t=Sticky
setfacl -m u:netops:rx /data/file && getfacl /data/file   # ACL 精细授权
chattr +i /path                   # 不可变位（连 root 都不能改）

# SSH 密钥登录三件套
chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys
restorecon -Rv ~/.ssh             # ★ 标签必须是 ssh_home_t，否则静默失败
```
**两种"拒绝"的区别**：`chattr +i` → `Operation not permitted`（内核位拒绝）；ACL/权限 → `Permission denied`（DAC 拒绝）。

---

## 四、存储与 LVM

```bash
lsblk / blkid / df -hT / findmnt          # 认盘、看 UUID、看类型、看挂载树
pvcreate /dev/sdb                          # 物理卷
vgcreate data_vg /dev/sdb /dev/sdc         # 卷组（跨盘池）
lvcreate -n data_lv -L 20G data_vg         # 逻辑卷
mkfs.xfs /dev/data_vg/data_lv              # 格式化
mount /dev/data_vg/data_lv /data           # 挂载

vgextend data_vg /dev/sdd                  # 扩池（池子见底的第一解法）
lvextend -r -L +5G /dev/data_vg/data_lv    # 在线扩容（-r 连带 resize2fs/xfs_growfs）
lvs -o +devices / lvdisplay -m             # 看 extent 落在哪块盘（linear）

lvcreate -s -n data_snap -L 2G /dev/data_vg/data_lv   # 快照（COW）
umount /data && umount /dev/data_vg/data_snap          # merge 前两边都要卸
lvconvert --merge /dev/data_vg/data_snap               # 回滚

UUID=xxxx /data xfs defaults 0 0           # fstab 用 UUID 最稳
mount -a && echo $?                        # ★ 黄金验证法：改完 fstab 必做
```
**碰到"差 1 个 extent"**：`lvextend -L` 的余粮卡在 VG 边界 → **先 `vgextend` 扩池**，再扩容（判据在余粮，不在写法）。

---

## 五、systemd

```bash
systemctl daemon-reload            # ★ 改完 unit 必做（systemd 有缓存）
systemctl enable --now myapp       # enable = 建 symlink 到 target.wants/
systemctl status myapp --no-pager  # 判决书在 status= 那行
journalctl -u myapp -n 50 -f       # 看日志
systemd-analyze blame              # 谁拖慢了开机（统计 boot 后所有启动活动）
systemd-analyze critical-chain     # 关键启动链
```
**unit 三段骨架**：`[Unit]` 描述与依赖（`After=`）｜`[Service]` 启动与自愈（`Type`/`ExecStart`/`Restart`）｜`[Install]` 挂靠 target（`WantedBy=`）

**Restart 两条规则**：`stop` 不插手（管理员意志）｜进程暴毙必救（`kill -9` 后自动换 PID）

**timer**：`OnCalendar=*-*-* 02:30:00`（日历式）/ `OnBootSec=` + `OnUnitActiveSec=`（单调式）｜**`enable` 的对象是 `.timer` 不是 `.service`**｜`AccuracySec` 默认 1min（触发会浮动，不是准点）

**SELinux 三部曲（放对目录 → mv 会带标签走 → `restorecon` 终审）**：
```bash
ls -Z /usr/local/bin/myapp.sh   # 看标签
restorecon -v /usr/local/bin/myapp.sh   # Relabeled admin_home_t → bin_t
```
**给 systemd 用的文件放对地方**：脚本 `/usr/local/bin`（bin_t）、数据 `/var/backups`、日志 `/var/log`。

---

## 六、网络

```bash
# 认门牌（配置层 vs 内核层，两个视角）
nmcli device status / nmcli connection show
ip -br a / ip r / ip -br link

# 配置落盘位置（openEuler 走 ifcfg-rh 插件，不是 keyfile）
/etc/sysconfig/network-scripts/ifcfg-ens33     # BOOTPROTO/IPADDR/PREFIX/GATEWAY/DNS1/ONBOOT
nmcli con reload && nmcli con up ens33          # 手改文件后（restart network 已失效）

# 改地址（+追加 / -删除 / 不加符号=覆盖！）
sudo nmcli con mod ens33 +ipv4.addresses 192.168.252.188/24
sudo nmcli device reapply ens33                  # ★ 热生效，不断 SSH
ip -br a show ens33                              # 内核层复核
# 多 IP 落盘写法：IPADDR1= / PREFIX1=

# 看监听
sudo ss -tlnp                                    # -t TCP -l 监听 -n 不解析 -p 进程
```
**路由读法**：`proto static` = 配置写死｜`proto kernel` = 内核按 IP+掩码自动生成｜`scope link` = 直连不走网关｜`metric` 小者优先。

**防火墙（firewalld）**：
```bash
sudo firewall-cmd --state / --get-active-zones   # running / public+接口
sudo firewall-cmd --list-all                     # 运行时账本
sudo firewall-cmd --list-all --permanent         # 永久账本  ★ 两本账必须一致
sudo firewall-cmd --permanent --add-port=8080/tcp && sudo firewall-cmd --reload   # ★ 顺序不能反
sudo firewall-cmd --permanent --remove-port=8080/tcp && sudo firewall-cmd --reload
```
- **区域 = 信任级别**：`trusted` > `home`/`work`/`internal` > `public`/`external` > `dmz` > `block` > `drop`
- `--add-port` 不加 `--permanent` = **临时通行证**，reload/重启即作废且不会自己回来
- 改之前先看 `services:` 有没有 `ssh`，否则 reload 把自己关在门外

**SELinux 端口层**：
```bash
getenforce / getenforce 0|1                       # 状态 / 临时宽容（验完必须开回）
sudo semanage port -l | grep <端口号>              # ★ 按端口查，别按类型名猜
sudo semanage port -m -t http_port_t -p tcp 9001  # 改归属（抢用 -m；-a 会报 already defined）
sudo semanage port -d -t http_port_t -p tcp 9001  # 删自定义 → 回落预定义类型
sudo semanage port -l -C                          # ★ 只看"本地自定义"= 机器被人动过哪些端口
sudo ausearch -m avc -ts recent                   # SELinux 拒绝的审计记录
```
**核心认知**：SELinux 管**进程的域**不是端口 —— 同绑 8080，`python3`(unconfined_t) 随便绑，`nginx`(**httpd_t**) 要过白名单；`httpd_t` 允许 `http_port_t` + `http_cache_port_t`（8080 属后者）。

**nginx reload 机理**：`systemctl reload` = 给 master 发 **SIGHUP** → 新 worker 先起 → 老 worker `gracefully shutting down` → **master 不动、监听不断 = 零中断**；reload 失败不伤老服务。

---

## 七、坑清单（本周 20 条）

| # | 坑 | 一句话 |
|---|---|---|
| 1 | 改 unit 不 `daemon-reload` | 白改（systemd 有缓存） |
| 2 | 脚本没裸跑就交给 systemd | 先 `timeout 12` 验尸 |
| 3 | `mv` 搬文件带 SELinux 标签 | admin_home_t 跟着走；`restorecon` 才是终点 |
| 4 | 手工造 `authorized_keys` 不 restorecon | 标签非 `ssh_home_t` → 静默失败 |
| 5 | `Restart=always` 时状态是 `activating` | 不是 failed；判决书在 `status=203/EXEC` |
| 6 | `systemctl stop` 不触发 Restart | Restart 只管非正常死亡 |
| 7 | timer 默认 `AccuracySec=1min` | 触发时刻会浮动（实测间隔 102s） |
| 8 | `--add-port` 不带 `--permanent` | 临时通行证，reload/重启即失 |
| 9 | 先 reload 后 permanent | 顺序反了规则被清 |
| 10 | `+` / `-` / `=` 三兄弟 | `+ipv4.addresses` 追加、`-` 删除、**不加符号=覆盖（危险）** |
| 11 | `reapply` 被忽略 | 改完必须 `ip -br a` 复核内核层 |
| 12 | 只本机 ping 就下结论 | 必须从对端交叉验证 |
| 13 | `sudo cmd *` 通配符 | **`*` 由当前 shell 展开，sudo 不提权** → 用 `sudo sh -c '… *'` |
| 14 | `sudo cat > /etc/xx` | 重定向由当前 shell 执行 → 用 `sudo tee` |
| 15 | 只看运行时账本 | 两本账不一致 = 早晚出事 |
| 16 | `No route to host` 误判成网络故障 | 是 REJECT（有 ICMP 回执、毫秒级） |
| 17 | `nginx -t` 通过就以为没事 | 它看不见 SELinux |
| 18 | curl 不加 `-v` | 简略模式把各种失败都写成 `Couldn't connect to server`，丢分层线索 |
| 19 | 用 `grep <类型名>` 推白名单 | 会漏（`http_port_t` 匹配不到 `http_cache_port_t`）→ **按端口号查** |
| 20 | 实验后不留清理 | 加减对称；`semanage port -d` 归还抢来的端口 |
| 21 | 多窗口 history 互相覆盖 | 开 `shopt -s histappend` + `PROMPT_COMMAND="history -a"` |
| 22 | `systemctl restart network` | openEuler 24.03 已失效，用 `nmcli con reload/up` |

---

## 八、本周能写进简历的一句

> 熟悉 openEuler 24.03 LTS 的 dnf / rpm 软件管理、LVM 磁盘管理与在线扩容、systemd 服务编排，以及 nmcli 网络配置与防火墙策略管理。
