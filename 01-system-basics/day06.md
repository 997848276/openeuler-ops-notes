# openEuler 练手笔记 · 2026-09-15（DAY3 后半：提权链路 + sudo 最小权限 + SSH 密钥登录 + 日志审计）

> 机器：128（oe-base，openEuler 24.03 LTS，kernel 6.6.0-145.0.13.139.oe2403）
> 目标：DAY3 后半「权限与 SSH 加固」，四组 —— A 提权链路 / B sudo 白名单 / C 密钥登录 / D 日志审计
> 排期铁律：**改远程访问配置，先加后减、留一条活路**（`PermitRootLogin no` 的"减法"挪到下次）
> 风格：每条 = 命令 + 现象 + 为什么 + 类比（证据链）

---

## 一、提权链路：`su` 与 `sudo` 要的是"谁的密码"

```bash
# 当前是 root
su - devops                    # → 直接进 [devops@oe-base ~]$，没问任何密码
sudo -l                        # 让输 devops 的密码 → 连续两次 Sorry, try again.
```

**三种情况分清（背下来）**

| 动作 | 谁执行 | 要谁的密码 |
|---|---|---|
| `su - bob` | **root** | **不要**（root 万能） |
| `su - bob` | 普通用户 | 要 **bob** 的密码 |
| `sudo xxx` | 任意已授权用户 | 要 **自己** 的密码 |

> 本机现场 **三次验证**了这条规律：① `su - devops`（root 执行）免密 ② `sudo -l` 要 devops 自己的密码 ③ 在 netops 身份下跑 `su - netops` 仍被要密码。

**⭐ 为什么 `sudo` 要"自己"的密码**

sudo 的设计是**验证身份（你是本人吗），不是验证权限（你有没有权）** —— 你已在 `wheel` 组、权限早就有，现在只是证明"你是这个账号本人"。两个好处：

1. 不必把 root 密码发给所有人；
2. **每个人用自己的密码提权 → 审计日志能追到具体的人**。

> 这正是政企安全审计最看重的一点：**共享 root 密码 = 出事查不到人**。

**坑位（必记）**：忘了自己的密码 → 从 root 执行 `passwd <user>` 或 `echo 'Xxx@123' | passwd --stdin <user>` 重置。

**类比**：`su` 像拿着**主管的万能卡**直接刷门（卡在谁手里是关键）；`sudo` 像你**本来就有权进金库**，但每进一次都要在门口**按自己的指纹**签到 —— 门开了，记录也留下了"是谁、几点进的"。

---

## 二、先看现状：`sudo -l` 与 `/etc/sudoers.d/`

```bash
sudo -l                        # → (ALL) ALL   （来自 %wheel）
sudo ls -l /etc/sudoers.d/     # → total 0     （目录是空的，正好留给我们加规则）
```

- **`(ALL) ALL`** 读法：第一个 `(ALL)` = 可切换成**任意身份**；后面的 `ALL` = 可执行**任意命令**。这是 `%wheel` 组的默认待遇（近乎全权）。
- **`%wheel` 的 `%` = 组**（`wheel` 是 RHEL 系的 sudo 特权组）。
- **最佳实践：不直接编辑 `/etc/sudoers`**，而是往 `/etc/sudoers.d/` 放**独立文件** —— 可追溯、可单独删除、便于版本管理，出错影响面也小。

**`sudo -l` 里最值钱的三条 Defaults**

| 指令 | 作用 |
|---|---|
| `!visiblepw` | 输密码**不回显**（`!` 是"取消/否定"） |
| `env_reset` + `env_keep` | 默认清空继承的环境变量，只放行白名单（LANG/LC_* 保住中文）→ 防"环境污染"攻击 |
| ⭐ `secure_path` | sudo 期间**强制替换 PATH** → 防"路径劫持"。**实战症状：直接跑能用、加 `sudo` 报 command not found** |

**类比**：`/etc/sudoers` 是公司的**总规章**；`/etc/sudoers.d/` 是各部门的**补充条例活页夹** —— 往活页夹插一页，比在总规章上手改安全得多。

---

## 三、亲手写"最小权限"白名单（★ 今天最大的坑在这里）

新建受限用户 `netops`（不在任何特权组），密码 `Netops@123`，规则文件 `/etc/sudoers.d/netops-readonly`：

```bash
netops ALL=(ALL) NOPASSWD: /usr/sbin/ss -tlnp
netops ALL=(ALL) NOPASSWD: /usr/bin/systemctl status
```

```bash
chmod 440 /etc/sudoers.d/netops-readonly    # 权限必须收紧
visudo -c                                    # 语法校验（保命）
```

**规则五段式**

```
netops   ALL=   (ALL)   NOPASSWD:   /usr/sbin/ss -tlnp
 谁      从哪台  以谁的    不用密码        允许的具体命令
```

- **`chmod 440`**：sudo **拒绝**加载"组/其他人可写"的规则文件（否则等于谁都能给自己开 sudo），文件松了会报 `sudo: /etc/sudoers.d/xxx is world writable!`。
- **`visudo -c` 改完必跑**：语法错会让 **所有** sudo 失效 —— **包括你自己的**。
- **拆成两行**比写成一条好：一条错不连累另一条，`sudo -l` 也更清楚。

### ★★ 3.1 sudoers 路径坑：`sudo -l` 会"骗"你

**现象（极具误导性）**：规则里把命令写成 `/usr/bin/ss` 时——
- `sudo -l` **正常列出**那条 `NOPASSWD` 规则；
- 但**执行时却仍然要密码**。

**真相**：`ss` 实际在 **`/usr/sbin/ss`**，`/usr/bin/ss` **根本不存在**。

**为什么 `sudo -l` 看不出来**：sudo 只是**把规则文本念一遍给你看**，**并不校验命令是否真实存在**。

**排障信号（记死它）**

> **`sudo -l` 里明明有 `NOPASSWD`，执行时却要密码 → 第一个念头就是"命令路径对不上"。**

**预防**：写白名单前先用 **`command -v <命令>`** 拿真实绝对路径。

```bash
command -v ss          # netops 下可能空输出（普通用户 PATH 不含 /usr/sbin）
command -v systemctl   # → /usr/bin/systemctl （这条一直是对的）
```

**类比**：门禁卡上印的**房号必须是真实存在的房号**。印错了，这张卡"看起来有效"，门却永远不认。

### 3.2 验证"最小"：四条对照实验

| 命令 | 结果 | 说明 |
|---|---|---|
| `sudo ss -tlnp` | ✅ **免密** | 逐字命中规则 |
| `sudo systemctl status` | ✅ **免密** | 命中第二条 |
| `sudo ss -tulnp` | ❌ 要密码 | **参数不一致就不算同一条命令** |
| `sudo cat /etc/shadow` | ❌ 要密码 | 根本不在白名单 |

- `sudo` 是**先认证（要密码）、再判定授权**：先问你密码，输完才告诉你"不允许"。
- `NOPASSWD` **只对逐字匹配的那一条命令生效**，多一个参数都不行。

**类比**：门禁只放行"刷卡 + 按对楼层"，你按错楼层，照样不放行 —— 同一张卡，不同动作。

---

## 四、SSH 密钥登录：从生成到免密（★ 连过三坎）

### 4.1 配钥匙

```bash
ssh-keygen -t ed25519 -C "netops@oe-base"
ls -l  ~/.ssh/            # id_ed25519 411B/600   id_ed25519.pub 96B/644
ls -ld ~/.ssh/            # drwx------. 700
```

- **`-t ed25519`**：现代首选算法（比 RSA 更短、更安全）。
- **`-C "..."` 只是注释/备注**：本次误写成 `netops@ob-base` —— **完全不影响功能，认证时忽略注释**。
- **权限是 sshd 的硬要求**：私钥 **600**、公钥 **644**、`~/.ssh` 目录 **700**。私钥若"组/他人可读"，sshd 甩 `UNPROTECTED PRIVATE KEY FILE` 直接拒用。
- 输出里的**指纹 + randomart 像素图**：供人眼核对，可用 `ssh-keygen -lf ~/.ssh/id_ed25519.pub` 重算。

**类比**：**私钥 = 你家钥匙**（绝不能给别人）；**公钥 = 钥匙模子**（可以随便发，登记在门卫那儿）。

### ★ 4.2 坎一：首次连接被 `known_hosts` 卡住（TOFU）

```
The authenticity of host 'localhost (::1)' can't be established.
ED25519 key fingerprint is SHA256:JyWWRD7DIgnlp0f3l3m2jTUW5LfPHguZEXqgD0fDD8g.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
...
/usr/bin/ssh-copy-id: ERROR: Host key verification failed.
```

**三层解读**

1. **第一次**连 localhost，`~/.ssh/known_hosts` 里**没有它的"脸"** → ssh 要求**人工核对指纹**（**TOFU = Trust On First Use**，防中间人）。
2. 提示里那串指纹是**服务器（localhost 的 sshd）的主机公钥指纹**，**不是**你刚生成的 `id_ed25519` —— 一个"门"的指纹，一个"钥匙"的指纹。
3. 提示**出现两次** = `ssh-copy-id` 内部开了**两次 SSH 连接**（先探测服务器接受哪些钥匙，再真正写 `authorized_keys`）。

**★ 坑位（必记）**

> 那个 `Are you sure to continue (yes/no/[fingerprint])?` 的**默认动作是"放弃"** —— **直接回车 ≠ 同意**，必须**手工敲三个字母 `yes` 再回车**。

**两种修法**

```bash
# A. 正规：重跑，出现提示时手工输 yes
ssh-copy-id netops@localhost

# B. 跳过交互（脚本里的标准姿势）
ssh-copy-id -o StrictHostKeyChecking=accept-new netops@localhost
```

- `accept-new` = **首次连接自动接受并记入 known_hosts**；**之后指纹若变了仍会拒绝**（安全与便利兼顾）。

**落地物证**：成功后 `~/.ssh/known_hosts` 出现（权限 **600**），里面存的是**对方主机的公钥**。

- ⭐ **`known_hosts` 是"每个用户一份"**：netops 接受过 localhost 之后，**root 再连仍会单独弹一次 TOFU 提示**（`Warning: Permanently added 'localhost' …`）—— 因为 `/root/.ssh/known_hosts` 与 `/home/netops/.ssh/known_hosts` 是**两个独立文件**。

**延伸（面试常问）**：服务器**重装/换机**后指纹会变，ssh 甩出 `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` + `Host key verification failed` → 清旧脸：`ssh-keygen -R <host>`（**`-R` = remove**），再重连。

### ★★ 4.3 坎二：`ssh-copy-id` 的"启动钥匙困境"

```
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new key(s)
Authorized users only. All activities may be monitored and reported.
netops@localhost: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
```

**一眼诊断（比翻配置文件还快）**：括号里是**服务器允许的认证方法** —— `publickey, gssapi-keyex, gssapi-with-mic`，**唯独没有 `password`**。

> **方法列表里没有 `password` = 服务器根本没开密码认证**（`PasswordAuthentication no`，且 `keyboard-interactive` 也未列）。

**为什么必然失败**：`ssh-copy-id` 的原理是"**登录到对端，并在对端执行写入 `authorized_keys` 的动作**"。而公钥**尚未到位**时，唯一可用凭据就是**密码** → 密码一关，它就**没有"启动钥匙"**（鸡生蛋）。

**正解：就地装公钥**（人已在机器上，根本不必走 SSH 通道）

```bash
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
ls -ld /home/netops ~/.ssh ~/.ssh/authorized_keys    # 查"三道锁"
wc -l ~/.ssh/authorized_keys                         # 应为 1
```

**验证结果（08:04）**：`authorized_keys` **96 字节**（与 `id_ed25519.pub` 的 96 字节**完全一致**）、权限 **600**、家目录与 `.ssh` 均 **700**、`wc -l` = **1**。

**类比**：`ssh-copy-id` 像"**让门卫帮你把钥匙模子贴到门上**" —— 可你得**先进得了门**才指挥得动门卫。密码就是那把"进门的备用钥匙"；备用钥匙被收走了，就只能**自己动手直接贴**。

**运维落点（面试加分）**：新机/新用户要发钥匙、而目标机已禁密码时，常用三条路：
① **就地装**（有其它登录通道：VMware/云控制台、跳板机）② **带外通道**（`cloud-init` / Ansible / 镜像预置）③ 临时开密码 → 发完钥匙 → **立刻再关**。

### ★ 4.4 坎三：验证免密——**客户端身份**决定用哪把私钥

**反例（现场实测）**：**root** 身份跑 `ssh netops@localhost` → `Permission denied`

> 铁律：**SSH 客户端用哪把私钥，取决于"发起连接的那个本地用户"，跟"远端要登哪个用户名"无关。**
> root 执行 → 用的是 `/root/.ssh/...`，而 netops 的 `authorized_keys` 里只有 netops 自己的公钥 → 必然失败。

**正解：用 netops 身份发起 ssh。** 两种验证：

```bash
# ① 交互式（netops 身份）
ssh netops@localhost
# → 无 password: 提示，直接打印 Last login 并进入 [netops@oe-base ~]$   ✅

# ② ★ 决定性验证：BatchMode 杜绝一切交互提示（脚本里的标准姿势）
ssh -o BatchMode=yes netops@localhost hostname
# → oe-base                                                          ✅
```

> `BatchMode=yes` = **绝不弹任何交互提示**。密码的"机会"都不给它，**能通就只能是密钥**。

**为什么这次的证明比平时更强**：该机**密码认证已经关闭**（见第七节），**密钥是唯一凭据** —— 能进就是 100% 铁证。

---

## 五、日志审计：`/var/log/secure`（D 组）

> 认知：`/var/log/secure` 是 RHEL 系（含 openEuler）记录**认证 / 提权 / 会话**的专用日志。
> 行结构：`月 日 时:分:秒 主机名 程序[PID]: 内容`

### 5.1 ★★ sudo 审计：谁 + 何时 + 何地 + 干了什么

**格式（被授权）**
```
Sep 15 07:48:49 oe-base sudo[2503]: netops : TTY=pts/0 ; PWD=/home/netops ; USER=root ; COMMAND=/usr/sbin/ss -tlnp
```

**四要素**：`TTY` = 哪条终端｜`PWD` = 当时所在目录｜`USER` = 要切成谁｜`COMMAND` = 执行什么命令。**加上行首的用户名 + 时间戳** → 谁、何时、在哪、以什么身份、跑了什么，**全齐**。

**格式（被拒绝）**：多一句 **`command not allowed`**
```
Sep 15 07:36:23 oe-base sudo[2111]: netops : command not allowed ; TTY=pts/0 ; PWD=/home/netops ; USER=root ; COMMAND=/usr/sbin/ss -tlnp
```

**★ 一眼判断"这条 sudo 到底跑没跑成"**：看有没有这一行
```
pam_unix(sudo:session): session opened for user root(uid=0) by root(uid=1001)
```
- **有** → 授权通过、命令**真的执行了**
- **没有** → 被拦，只有日志记录、没有实际动作

### 5.2 ★★ 路径坑的"案发现场"：同一条命令，前后两态

| 时间 | 日志行 | 状态 |
|---|---|---|
| `07:36:23` | `netops : command not allowed ; … COMMAND=/usr/sbin/ss -tlnp` | ❌ **拒**（当时规则写的是 `/usr/bin/ss`，路径对不上） |
| `07:48:49` | `netops : TTY=pts/0 ; PWD=/home/netops ; USER=root ; COMMAND=/usr/sbin/ss -tlnp` + `pam_unix(sudo:session): session opened … by root(uid=1001)` | ✅ **放行**（路径改为 `/usr/sbin/ss`） |

**另两条被拒物证（白名单的边界）**
- `07:49:21 … command not allowed ; … COMMAND=/usr/sbin/ss -tulnp` → **参数不同 ≠ 同一条命令**
- `07:51:53 … command not allowed ; … COMMAND=/usr/bin/cat /etc/shadow` → 白名单之外
- `08:00:57 … netops : TTY=pts/0 ; … COMMAND=/usr/sbin/sshd -T` → **任何 sudo 尝试都会留痕**，与终端那句 "Sorry, … is not allowed" 互为印证

### 5.3 su 与 sudo 在日志里的"身份差异"（为什么 sudo 更可审计）

```
su[2044]:   pam_unix(su-l:session):  session opened for user netops(uid=1001) by root(uid=0)
sudo[2503]: pam_unix(sudo:session):  session opened for user root(uid=0)    by root(uid=1001)
```

- `su` 用 **`su-l:session`**，记的是**"谁把我切过去的"**（`by root`）
- `sudo` 记 **`by root(uid=1001)`** —— **保留了真实发起人 uid 1001（netops）**，只是"以 root 身份执行"

> 这就是 **"sudo 要自己的密码"的最终目的**：日志里追得到**具体的人**。

### 5.4 ★ 密码策略与防爆破的留痕

```
passwd[2012]: pam_unix(passwd:chauthtok): new password not acceptable    ← 弱口令被策略拒
passwd[2016]: pam_unix(passwd:chauthtok): password changed for netops    ← 合规后通过
```
→ **`pwquality` 密码复杂度策略在起作用**（等保要求项）

```
sudo[1878]: devops : 2 incorrect password attempts ; TTY=pts/0 ; PWD=/home/devops ; USER=root ; COMMAND=list
pam_faillock(sudo:auth): Consecutive login failures for user devops account temporarily locked
```
→ **★ 连续输错 → `pam_faillock` 临时锁定账号**（防爆破，等保要求项）；`COMMAND=list` 即 `sudo -l`

### 5.5 会话、建号与交叉验证

```
useradd[2004]: new user: name=netops, UID=1001, GID=1001, home=/home/netops, shell=/bin/bash, from=/dev/pts/0
sshd[2228]: pam_unix(sshd:session): session opened for user root(uid=0)                          ← 外部(2222) root 密钥登录
sshd[2771]: pam_unix(sshd:session): session opened for user netops(uid=1001) by netops(uid=0)    ← 本地免密登录
```

- `useradd` 的 **`from=/dev/pts/0`**：连"从哪个终端建的号"都留痕。
- **交叉验证**：`who ; w` → **2 条，均为 `root`，都来自 `192.168.252.1`（Windows 宿主）**，`pts/0` 登录 `07:20`、`pts/1` 登录 `08:03` —— 与上面 sshd 的 `session opened` 行（`07:20:54` / `08:03:50`）**分钟级对齐**。
  > `who` 只列**登录会话**；`su`/`sudo` 出来的不算 —— 这也解释了 MOTD 里 `Users online: 3` 会回落成 2。

---

## 六、★ 谜题：`Accepted` 为什么不在 `/var/log/secure`

**现象**：统计命令空手而归
```bash
grep -oP 'Accepted \w+ for \K\w+' /var/log/secure | sort | uniq -c   # → 无输出
grep -c 'Accepted' /var/log/secure ; grep -c 'Failed password' /var/log/secure   # → 0 ; 0
```

**★ 根因（已确诊）**
```bash
sshd -T | grep -Ei 'syslogfacility|loglevel'
# → loglevel VERBOSE
#    syslogfacility AUTH
```
**本机 `SyslogFacility` 生效值是 `AUTH`**（OpenSSH 传统默认是 **`AUTHPRIV`**）。而 **syslog 是按 facility 分流的**：

| 日志来源 | facility | 落到哪 |
|---|---|---|
| **sshd 自己**的 `Accepted` / `Failed password` / `Connection from` | 跟随 `SyslogFacility` → **`AUTH`** | **`/var/log/messages`** |
| **PAM 模块**（`pam_unix` / `pam_faillock`）的会话与认证消息 | **`AUTHPRIV`**（模块内固定） | **`/var/log/secure`** |

→ 解释了"secure 里只有 PAM 的会话/提权记录、却没有 sshd 的登录明细"；也解释了 §一 为什么用 **`journalctl -u sshd`** 才查到登录记录：**journald 不分 facility、全收**（`journalctl -u sshd --since today | grep -c Accepted` → **18**）。

**换源头验证（08:32）**
```
Sep 15 08:06:46 oe-base sshd[2846]: Accepted key ED25519 SHA256:RgK1Mr8HgavhmbvOM8f5haBSf5Y2AUCWjBlp0/MxB00 found at /home/netops/.ssh/authorized_keys:1
Sep 15 08:06:46 oe-base sshd[2846]: Accepted publickey for netops from ::1 port 59606 ssh2: ED25519 SHA256:RgK1Mr8HgavhmbvOM8f5haBSf5Y2AUCWjBlp0/MxB00
```
- `Accepted key … **found at /home/netops/.ssh/authorized_keys:1**` —— **服务器亲口确认"在第 1 行找到了这把公钥"**；`:1` 正好对上前面 `wc -l` = **1**（唯一一行）。
- `from **::1** port 59606` —— 来源是 **IPv6 本地回环**（本地 ssh 场景）。
- ⭐ **指纹三方闭环**：`ssh-keygen` 生成时 → `authorized_keys` 里 → **日志里**，三处完全一致。
- 这两行 `Accepted key … :N` 是 **`loglevel VERBOSE`** 的功劳（默认 `INFO` 只记 `Accepted publickey`，不记"匹配到哪把钥匙、在第几行"）。

**配置文件确认**
```
/etc/ssh/sshd_config:34:#SyslogFacility AUTH      ← 自带的注释示例
/etc/ssh/sshd_config:35:SyslogFacility AUTH       ← 真正生效的那条（无 #）
```
⚠️ **经典陷阱**：OpenSSH 自带示例注释写的是 `#SyslogFacility AUTH`，但**编译默认值是 `AUTHPRIV`** —— 照着示例"取消注释"，就会把 sshd 日志从 `/var/log/secure` 挪到 `/var/log/messages`。

**想确认"发行版自带"还是"本地改过"**：
```bash
rpm -V openssh-server | grep sshd_config
# 有输出（形如 S.5....T.  c /etc/ssh/sshd_config）→ 文件被改过（c = config 类）
# 无输出 → 就是发行版自带
```

**★ 附赠一个"假阳性"教训**：`grep -oP 'from \K[0-9a-f.:]+'` 曾统计出诡异的 **36 个 `d`**，真身是
```
polkitd[1281]: Loading rules from directory /etc/polkit-1/rules.d
```
—— 正则抓到了 "from **d**irectory" 里的那个 `d`（只有 `d` 是十六进制字符）。
> **教训：出统计前先 `head` 抽样看原文，别让正则替你下结论。**
> （顺带看到主机名的演变：`bogon`(9/2) → `localhost`(9/5) → `oe-base`。）

> **运维铁律**：**"查不到"不等于"没发生"** —— 先确认"证据该在哪个文件 / 哪个 facility"，再下结论。

---

## 七、SSH 加固现状确诊：`sshd -T`

```bash
sshd -T | grep -Ei 'passwordauth|kbdinteractive|permitrootlogin|pubkeyauth'
```
（`-T` = 导出**最终生效配置**，不受多份 conf 叠加影响，比翻文件权威。）

**回显（08:07 · 128）**
```
permitrootlogin without-password
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
pubkeyauthoptions none
```

| 指令 | 生效值 | 含义 |
|---|---|---|
| `pubkeyauthentication` | `yes` | 密钥登录可用 ✅（免密方案的地基） |
| `passwordauthentication` | **`no`** | **密码登录已关** —— 印证 §四"方法列表里没有 `password`"；**"减法"的一半 9/14 已完成** |
| `kbdinteractiveauthentication` | `no` | 交互式键盘认证也关 → 正是方法列表里没有 `keyboard-interactive` 的原因 |
| `permitrootlogin` | **`without-password`** | ⭐ **中间态**：root 可登录、但**只能用密钥** |
| `pubkeyauthoptions` | `none` | 无附加限制 |

**`PermitRootLogin` 三个档位（面试常问）**

| 取值 | 含义 |
|---|---|
| `yes` | root 可用**任意**方式登录（含密码）—— 最危险 |
| **`without-password`**（别名 `prohibit-password`） | root 可登录，但**只能用密钥等非密码方式** ← 本机现状 |
| `no` | **完全禁止** root 远程登录 |

**⚠️ 为什么不能直接改成 `no`（"先加后减"的具体含义）**
本机现为 `without-password` → 平时从 Windows 经 **2222** 端口登进来的，很可能**正是 root 的密钥会话**（`who` 的证据：两条 root 会话都来自 `192.168.252.1`）。若直接 `PermitRootLogin no` 又**没有另一个"能密钥登录 + 能 sudo 管理"的账号**顶上，就只剩 VMware 控制台一条路了。

**"减法"正确顺序**
1. **先加**：给 `devops`（在 `wheel` 组、有 sudo 权）也配好**密钥登录** → 验证它能从外部免密登入且能 `sudo`
2. **再减**：`PermitRootLogin no` → 校验语法 → `systemctl restart sshd`
3. **验证**：用 **devops** 从 Windows 新开会话登入 + 跑 `sudo -l`；同时确认 **root 已被拒**
4. **保底**：全程留着 VMware 控制台（它**不走 sshd**，永远能进）

**类比**：换门锁之前，先确认**新钥匙已在手里、且试开过一次**；不然锁一换，你就得砸门。

---

## 八、今日踩坑清单

| 踩坑 | 原因 | 正确做法 |
|---|---|---|
| 以为 `su - devops` 免密是"配置好了" | 当时是 **root** 身份，root 切谁都不用密码 | 分清"谁执行 su"三种情况 |
| `sudo sshd -T` 被白名单拦下 | `sshd -T` 需 **root**；用 netops 跑必被拒 | 切回 root 直接跑（root 不需 sudo） |
| `ssh-copy-id` 提示后直接回车 → `Host key verification failed` | 该提示的**默认动作是"放弃"** | 必须**手输 `yes` 再回车** |
| 以为 SSH 装了公钥就能连 | **客户端身份**决定用哪把私钥 | root 跑 `ssh netops@…` 必失败；要用 netops 身份发起 |
| `sudo -l` 显示 `NOPASSWD` 却仍要密码 | **命令路径写错**（`/usr/bin/ss` ≠ `/usr/sbin/ss`），sudo **不校验路径** | 写规则前 `command -v <命令>` 核实绝对路径 |
| `grep -oP 'from \K…'` 统计出 36 个 `d` | 正则太宽松，匹配到 "from **d**irectory" | 出统计前先 `head` 抽样看原文 |
| 以为 `/var/log/secure` 该有登录记录 | 本机 `SyslogFacility AUTH` → sshd 日志在 **`/var/log/messages`** | 先确认证据落在哪个 facility / 文件 |

**工具箱通用铁律**

> ① **`sudo -l` 里有 `NOPASSWD` 却要密码 → 先查路径。**
> ② **查不到日志 ≠ 没发生 → 先确认 facility / 文件。**
> ③ **出统计前先抽样 → 别让正则替你下结论。**
> ④ **改远程访问配置先加后减 → 永远留一条活路。**

---

## 九、面试一句话（直接当素材）

- **"`su` 和 `sudo` 的区别？"** → `su` 是切换身份（root 切谁都不需密码；普通用户要目标用户密码）；`sudo` 是**以本人身份、经授权后**执行特权命令，**要的是自己的密码** —— 因为 sudo 验证的是**身份**而非权限，好处是**审计能追到人**（政企最看重）。授权规则放 `/etc/sudoers.d/`（独立文件、`chmod 440`、改完 `visudo -c`），权限要**最小化且逐字精确**（参数不同就不匹配）。

- **"sudo 配了却不生效，怎么查？"** → 高频坑是**规则里命令路径写错** —— `sudo -l` 会照念规则文本却不校验路径，表现为"列着 `NOPASSWD` 却仍要密码"；写规则前用 `command -v` 核实绝对路径。另外用 `pam_unix(sudo:session): session opened` 判断命令是否真的执行了。

- **"怎么给服务器配免密登录？"** → `ssh-keygen -t ed25519` 生成密钥（私钥 600、`~/.ssh` 700、公钥 644），`ssh-copy-id` 把公钥写入对端 `authorized_keys`；首次连接走 **TOFU** 人工核对主机指纹（**提示默认是放弃，须手输 `yes`**），指纹存 `known_hosts`（**每个用户一份**），换机后指纹变化用 `ssh-keygen -R` 清理；验证用 `ssh -o BatchMode=yes` 杜绝交互提示。

- **"对端已禁用密码，怎么发新钥匙？"** → `ssh-copy-id` **依赖密码认证作为引导**，对端禁密码后必然失败（信号：认证方法列表里**没有 `password`**）；此时走**就地装公钥**（`cat .pub >> authorized_keys` + `chmod 600`，并确保家目录、`.ssh`、`authorized_keys` 三者都不可被组/他人写），或走带外通道（控制台 / `cloud-init` / Ansible），或临时开密码 → 发完立刻关。

- **"日志怎么看权限审计？"** → `/var/log/secure` 的 **sudo 审计四要素**：`TTY`（哪条终端）、`PWD`（当时目录）、`USER`（切成谁）、`COMMAND`（跑什么），加上行首的用户名与时间戳，就是完整的**谁-何时-何地-干了什么**；被拒条目多一句 **`command not allowed`**。**`sudo` 日志会保留真实发起人**（netops 的 uid 1001），这就是"每人用自己的密码提权"的价值。日志里还能看到 `pwquality` 拒弱口令、`pam_faillock` 连续失败锁号、`useradd` 建号（含 `from=` 终端）等留痕。

- **"日志里查不到某条记录怎么办？"** → **"查不到"不等于"没发生"**：syslog **按 facility 分流** —— sshd 自身日志跟随 `SyslogFacility`（本机 `AUTH` → `/var/log/messages`），PAM 模块固定用 `AUTHPRIV`（→ `/var/log/secure`），所以"登录明细"与"提权记录"天然分居两个文件；`journalctl` 不分 facility 全收。用 `sshd -T | grep -Ei 'syslogfacility|loglevel'` 看生效值，`rpm -V` 判断配置是自带还是被改。

- **"SSH 怎么加固？"** → `sshd -T` 一屏看全生效值：`PasswordAuthentication no`（禁密码）、`PubkeyAuthentication yes`、`PermitRootLogin` 三档（`yes` / `without-password` / `no`）；**改远程访问配置永远"先加后减"** —— 禁用 root 远程前，必须先确保有一个**能密钥登录、能 sudo 的普通账号**作为替代入口，并保留**控制台**这条带外后路。
