# openEuler 练手笔记 · 2026-09-16（DAY3 收官：SSH「减法」—— `PermitRootLogin no`）

> 机器：128（oe-base，openEuler 24.03 LTS，kernel 6.6.0-145.0.13.139.oe2403，SELinux **Enforcing**）
> 目标：把 9/15 留下的最后一刀落地 —— **`PermitRootLogin no`**（9/14-9/15 已完成"加法"：密钥登录 / sudo 最小权限 / 日志审计）
> 铁律：**先加后减、留一条活路**。本次的"加"= 给 `devops` 配密钥登录 + 验证能 sudo
> 风格：每条 = 命令 + 现象 + 为什么 + 类比（证据链）

---

## 一、块 0 体检：**发现"备胎根本没装"**（今天最大的收获来自"没动手之前"）

```bash
whoami; id
sshd -T | grep -Ei 'permitrootlogin|pubkeyauthentication|passwordauthentication|kbdinteractive|^port'
ls -l /home/devops/.ssh/ 2>/dev/null && echo "--- authorized_keys 行数 ↓ ---" && wc -l /home/devops/.ssh/authorized_keys 2>/dev/null
ls -l /etc/sudoers.d/
cat /etc/sudoers.d/* 2>/dev/null
getenforce
```

**回显**
```
[root@oe-base ~]# whoami; id
root
uid=0(root) gid=0(root) groups=0(root) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023

port 22
port 2222
permitrootlogin without-password
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no

[root@oe-base ~]# ls -l /home/devops/.ssh/ 2>/dev/null && echo "--- authorized_keys 行数 ↓ ---" && wc -l ...
                                       ← ★ 一行输出都没有！

total 4
-r--r----- 1 root root 99 Sep 15 07:48 netops-readonly      ← 只有 netops 的

netops ALL=(ALL) NOPASSWD:/usr/sbin/ss -tlnp
netops ALL=(ALL) NOPASSWD: /usr/bin/systemctl status

Enforcing
```

**★★ 三条硬结论**
| # | 发现 | 后果 |
|---|---|---|
| ① | `whoami` = **`root`** | 当前会话就是 **root 密钥会话** → 改完一旦断线就回不来（只剩 VMware 控制台） |
| ② | **`/home/devops/.ssh/` 不存在** | ★ **`devops` 没有 `authorized_keys` → 备胎没装！** |
| ③ | `sudoers.d` 只有 `netops-readonly` | `devops` 的提权**待确认** |

**★ 重大修正**：原以为"加法（密钥登录）9/14-9/15 已完成" —— **实际只配了 `root` 的**（正因如此当时才能 root 免密登进来），**`devops` 的从来没配过**。

> ### ★ 这就是"先加后减"存在的意义
> 它防的不是流程洁癖，**它防的正是"你以为加完了，其实没加"**。
> 而唯一能证明"加完了"的方式是 —— **从外部、用新账号、真的登进来过一次。没登过，就不算。**

**★ 隐蔽坑：`2>/dev/null` 把关键证据一起吞掉了**
```bash
ls -l /home/devops/.ssh/ 2>/dev/null && echo ...    # ✗ 目录不存在时"静默"，看起来像没问题
ls -ld /home/devops /home/devops/.ssh               # ✓ 让报错露出来，报错本身就是证据
```
> **排查时慎用 `2>/dev/null`** —— 它让"失败"和"没输出"长得一模一样。

---

## 二、块 0.5 摸家底：备胎只差"装公钥"

```bash
id devops
ls -ld /home/devops
sudo -l -U devops
grep -nE 'wheel' /etc/sudoers
```

**回显**
```
uid=1000(devops) gid=1000(devops) groups=1000(devops),10(wheel),3000(appgrp)
65536 1 /home/devops/

User devops may run the following commands on oe-base:
    (ALL) ALL                                   ← ★ 全权 sudo ✓

106:## Allows people in group wheel to run all commands
107:%wheel  ALL=(ALL)       ALL                 ← 未注释 = devops 全权的来源（% = 组）
110:# %wheel        ALL=(ALL)       NOPASSWD: ALL  ← 注释掉了 = sudo 要输密码
```

| 项 | 状态 |
|---|---|
| `devops` 账号 | ✅ uid=1000 |
| 在 `wheel` 组 | ✅ |
| **sudo 全权** | ✅ `(ALL) ALL` |
| sudo 需密码 | ✅ 需要（**正常，别去打开 110 行** —— 密码是第二因子） |
| 家目录 | ✅ |
| **`authorized_keys`** | ❌ **唯一缺的** |

> **★ 好用的命令：`sudo -l -U <用户>`** —— root 可**代查**任意用户的 sudo 权限，不用切过去（核查/审计神器）。

---

## 三、块 1：给 `devops` 装公钥（★ SELinux 标签是关键）

**★ 私钥在客户端生成（Xshell「用户密钥管理者」→ Ed25519）**，只把**公钥**带上服务器。
> 理由：① **生产规范 —— 私钥永不离开客户端**；② 若在服务器生成再 `cat` 出来粘贴，**私钥换行极易损坏**，之后会对着"登录失败"白查半小时。

```bash
install -d -m 700 -o devops -g devops /home/devops/.ssh

cat > /home/devops/.ssh/authorized_keys <<'EOF'
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGonR1BjC/CYwx6tdP2CLt9g6wn9XPKCbMkT1YNM0HTQ leihao-win
EOF

chown devops:devops /home/devops/.ssh/authorized_keys
chmod 600 /home/devops/.ssh/authorized_keys

restorecon -Rv /home/devops/.ssh        # ★ Enforcing 下必做
ls -lZ /home/devops/.ssh
```

**回显**
```
-rw-------. 1 devops devops unconfined_u:object_r:ssh_home_t:s0 92 Sep 16 11:00 authorized_keys
```

**★ `ls -lZ` 三段式标签读法**：`SELinux用户:角色:类型`
| 段 | 值 | 说明 |
|---|---|---|
| user | `unconfined_u` | 非受限期用户 |
| role | `object_r` | 文件一律 `object_r`（进程才用具体 role） |
| **type** | **`ssh_home_t`** ✅ | ★ **sshd 只认这个类型** |

**为什么 `restorecon` 是这一步的灵魂**
手工 `cat >` 造的文件，标签**继承父目录**（`/home/devops` 是 `user_home_t`）→ 新文件也成 `user_home_t` → **sshd 读不了，而且不会大声报错**，只会"假装没看见你的公钥"。
**静默失败，是最难查的一类故障。** `restorecon` 按策略打"政策默认值"（比 `chcon` 持久，不会被 relabel 冲掉）。

**★ 权限"三件套"必须同时正确**（sshd 逐级检查，任一被组/他人可写就拒用密钥）：
`.ssh` **700** + `authorized_keys` **600** + 家目录**不可被组/他人写**

**★ 块 1-C 新会话验证（走 2222）**
```
[devops@oe-base ~]$ whoami
devops                       ← ① 免密登进来了（公钥生效）
[devops@oe-base ~]$ sudo whoami
[sudo] password for devops:
root                         ← ② 提权也通
```

> **★★ 「加完了」的判据是"走通一次"，不是"文件写对了"** —— 必须**同时**满足：
> ① **从外部**发起 ② **用新账号** ③ **真的登进来 + 真的提到权**。

---

## 四、块 2：那一刀 `PermitRootLogin no`

**2-A 备份 + 勘察**
```bash
cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.20260916
grep -nE '^#?PermitRootLogin' /etc/ssh/sshd_config
ls -l /etc/ssh/sshd_config.d/ 2>/dev/null
```
```
41:#PermitRootLogin prohibit-password      ← 有这一行，但被注释！
（后两条无输出：无 drop-in 目录）
```
> ★ **这一行是注释的** → 说明之前的 `without-password` **走的是 sshd 的"默认值"**。所以这次改动其实是"**把默认行为变成明确的拒绝**"。
> **勘察为什么必要**：若这一行**不存在**，`sed` 会**静默地什么都不改**（无匹配 → 无报错 → 你以为改了）。

**2-B 改写**
```bash
sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
```
| 片段 | 作用 |
|---|---|
| `-i` | **原地**改文件（不加只打印到屏幕 = 白改） |
| `^#\?` | 行首可有可无的 `#` → **注释版和启用版都能命中** |
| `.*` | 该行余下内容全吃掉 |

**2-C 回读 + 语法校验**
```bash
grep -n 'PermitRootLogin' /etc/ssh/sshd_config
```
```
41:PermitRootLogin no
91:# the setting of "PermitRootLogin prohibit-password".
```
> ★ **第 91 行是"注释文字里提到了这个词"，不是指令。** `grep 关键词` 会把"注释中提到该词"的行一并捞出来 —— **别把注释当配置**。
> 所以 2-A 用 `^#?PermitRootLogin`（**行首锚定**）才精确；宽泛 grep 回读时，要自己分辨指令行与注释行。

```bash
sshd -t && echo "=== 语法 OK ==="
# === 语法 OK ===
```

**★★ 2-D 预演：不重启就看见"最终生效值"（本节最值钱的技巧）**
```bash
sshd -T | grep permitrootlogin
# permitrootlogin no
```
> `sshd -T` = 让 sshd **按配置规则解析出最终生效值并打印**，**不启动、不重启、不断现有连接**。
> **= 换锁之前先试钥匙。** 显示 `no` 才敢重启；还是 `without-password` → **说明改动没被读到**（`sed` 没匹配上 / 优先级问题），**此时先别重启**。

| 命令 | 干什么 | 回答的问题 |
|---|---|---|
| `sshd -t` | **语法校验** | 配置写得**对不对** |
| **`sshd -T`** | **导出最终生效值** | 配置**生效成什么样** |

> **语法对 ≠ 生效对**（`sed` 一行没匹配上，语法依然合法但值没变）→ **两个都要跑。**

**2-E 重启 + 认读**
```bash
systemctl restart sshd
systemctl status sshd --no-pager | head -5
```
```
● sshd.service - OpenSSH server daemon
     Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-09-16 11:05:32 CST; 6s ago
```
> ⚠️ **`restart sshd` 不断开已有连接**（老会话照旧），**只有新连接才用新配置** → **真正的验收必须新开会话**。

---

## 五、块 3：双向验证 —— ★★★ 客户端报错 ≠ 服务端真相

**3-A 新会话 · `devops` 免密登入** ✅
```
[devops@oe-base ~]$ whoami
devops
```

**3-B 新会话 · `root` 登录 → 被拒** ✅
**Xshell 弹窗（★ 这句话是"误导"）**
```
远程主机:   192.168.252.128:2222 (ob)
登录名:     root
服务器类型: SSH2, OpenSSH_9.3
所选用户密钥未在远程主机上注册。请再试一次。
```

**3-C 服务端日志（★ 真相在这里）**
```bash
grep -i sshd /var/log/messages | tail -10
```
```
Sep 16 11:08:21 oe-base sshd[2014]: Accepted key ED25519 SHA256:Fm3PdZAnsaDzUAOkofmxTmilPqXo1Yev+ZcowaWzgLY found at /root/.ssh/authorized_keys:1
Sep 16 11:08:21 oe-base sshd[2014]: Postponed publickey for root from 192.168.252.1 port 1055 ssh2 [preauth]
Sep 16 11:08:21 oe-base sshd[2014]: Accepted key ED25519 SHA256:Fm3Pd… found at /root/.ssh/authorized_keys:1
Sep 16 11:08:21 oe-base sshd[2014]: ROOT LOGIN REFUSED FROM 192.168.252.1 port 1055
Sep 16 11:08:21 oe-base sshd[2014]: Failed publickey for root from 192.168.252.1 port 1055 ssh2: ED25519 SHA256:Fm3Pd…
Sep 16 11:08:21 oe-base sshd[2014]: ROOT LOGIN REFUSED FROM 192.168.252.1 port 1055 [preauth]
Sep 16 11:08:22 oe-base sshd[2014]: refusing previously-used ED25519 key [preauth]
Sep 16 11:08:23 oe-base sshd[2014]: refusing previously-used ED25519 key [preauth]
Sep 16 11:08:32 oe-base sshd[2014]: error: Received disconnect from 192.168.252.1 port 1055:0: [preauth]
Sep 16 11:08:32 oe-base sshd[2014]: Disconnected from authenticating user root 192.168.252.1 port 1055 [preauth]
```

**逐条读**
| 日志行 | 含义 |
|---|---|
| `Accepted key … found at /root/.ssh/authorized_keys:1` | ★ **服务端认得这把钥匙**，还指出"在第 1 行找到了" |
| `Postponed publickey for root` | 公钥先"搁置"记下，等后续认证流程 |
| **`ROOT LOGIN REFUSED FROM …`** | ★★ **真相：钥匙没问题，是"root 这个用户不允许登录"** —— `PermitRootLogin no` 的专属日志 |
| `Failed publickey for root` | 这次公钥认证**记为失败**（原因不是钥匙错，而是用户被拒） |
| `refusing previously-used ED25519 key` | ★ 客户端**自动重试时又用了同一把钥匙** → OpenSSH 拒绝"同一次预认证里重复使用同一把公钥"，**不是"钥匙坏了"** |
| `Disconnected from authenticating user root … **[preauth]**` | ★ **`[preauth]` = 认证尚未完成就断开** → **没走到"认证成功之后"**，在认证阶段就被拦下 |

**★★★ 客户端 vs 服务端**
| 谁 | 说了什么 | 真假 |
|---|---|---|
| **Xshell（客户端）** | "**所选用户密钥未在远程主机上注册**" | ❌ **误导** |
| **sshd（服务端）** | "**Accepted key … found at /root/.ssh/authorized_keys:1**" + "**ROOT LOGIN REFUSED**" | ✅ **真相** |

> **为什么客户端会冤枉钥匙**：客户端**只知道"认证失败了"**，**无法区分**"服务器上没这把钥匙"还是"服务器压根不许这个用户登录"（还可能被 `DenyUsers` / `AllowUsers` / 账号锁定 / 密码过期等原因拒掉）。
> 于是它**给出了一个最常见的猜测** —— 而**猜测往往就是误导**。
>
> ### ★ 排障铁律：客户端只管猜，服务端才有真相。凡是认证类故障，必须看服务端日志。

**★ 指纹三方闭环**：`ssh-keygen` 生成时 → `authorized_keys` 里 → **服务端日志**里（`SHA256:Fm3PdZn…WzgLY`），三处一致。
**★ 目标达成的三条独立证据**：`sshd -T | grep permitrootlogin` = **`no`** ＋ **`ROOT LOGIN REFUSED`** ＋ **devops 免密登入成功**。

---

## 六、今日踩坑清单

| 踩坑 | 原因 | 正确做法 |
|---|---|---|
| 以为"加法已完成"，其实只配了 root 的密钥 | 没区分"谁的密钥配了" | **动手前先体检**，逐个账号核实 |
| `ls -l /home/devops/.ssh/ 2>/dev/null` 静默无输出 | **`2>/dev/null` 把"目录不存在"一起吞了** | 排查时慎用 `2>/dev/null`，让报错露出来 |
| 手工建的 `authorized_keys` 标签是 `user_home_t` | 标签**继承父目录**，sshd 只认 `ssh_home_t` | **`restorecon -Rv <家目录>/.ssh`** + `ls -lZ` 复核 |
| `grep 'PermitRootLogin'` 多出一行第 91 行 | 那是**注释文字里提到了这个词** | 用 `^#?PermitRootLogin` **行首锚定**匹配指令 |
| 改完配置不放心 | `sshd -t` 只校验**语法**，不校验**生效值** | **再加 `sshd -T` 预演最终值**（不重启就能看） |
| Xshell 说"密钥未在远程主机上注册" | **客户端只能猜**，无法区分"没钥匙"和"用户被禁" | **看服务端日志**：`Accepted key … found at …` + `ROOT LOGIN REFUSED` |
| 日志里刷出 `refusing previously-used … key` | 客户端自动重试又用了同一把钥匙 | 不是钥匙坏，**忽略即可** |

**工具箱通用铁律（本次新增两条）**

> ① **`sudo -l` 里有 `NOPASSWD` 却要密码 → 先查路径。**（9/15）
> ② **查不到日志 ≠ 没发生 → 先确认 facility / 文件。**（9/15）
> ③ **出统计前先抽样 → 别让正则替你下结论。**（9/15）
> ④ **改远程访问配置先加后减 → 永远留一条活路。**（9/15）
> ⑤ ★ **"加完了"只有"从外部、用新账号、真的走通一次"才算数。**（9/16）
> ⑥ ★ **认证类故障，客户端提示只是猜测，服务端日志才是真相。**（9/16）

---

## 七、面试一句话（直接当素材）

- **"SSH 怎么加固？加固顺序是什么？"** → 先用 **`sshd -T`** 一屏看全生效值（它是"最终生效配置"的出口，不受多份 conf 叠加影响）：`PasswordAuthentication no` 禁密码、`PubkeyAuthentication yes`、`PermitRootLogin` 三档（`yes` / `without-password` / **`no`**）。**改远程访问配置永远"先加后减"**：禁用 root 远程之前，必须先确保有一个**能密钥登录、且能 sudo 的普通账号**作为替代入口，**并且从外部真的登进去验证过一次**；再改配置 → `sshd -t` 校验语法 → **`sshd -T` 预演生效值** → `systemctl restart sshd` → **新开会话**双向验证（新账号能进 / root 被拒）；全程保留**控制台**这条带外后路（它不走 sshd）。

- **"为什么不能直接 `PermitRootLogin no`？"** → 因为 `restart sshd` 不会断开已有连接，**你当时那条 root 会话还能用，会让你误以为没问题**；但**一旦断开就再也进不来**，只剩物理/虚拟控制台。所以必须先证明"替代路径走通过" —— 这就是"先加后减"的意义：**它防的正是"你以为加完了、其实没加"**。

- **"改完 sshd 配置怎么验证才放心？"** → 三道：**`sshd -t`** 校验语法（不过就绝不重启）→ **`sshd -T`** 预演最终生效值（**不重启、不断连接就能看见结果 = 换锁前先试钥匙**）→ **新开会话**双向验证。记住**语法对 ≠ 生效对**：`sed` 没匹配到任何行时配置语法依然合法，但值根本没变。

- **"SSH 登录失败，怎么定位？"** → **客户端提示不可信** —— 它只知道"认证失败"，无法区分"没有这把钥匙"和"这个用户被禁止登录"，于是给出最常见的猜测（如 Xshell 的"密钥未在远程主机上注册"）。**看服务端日志才有真相**：`Accepted key … found at <文件>:<行号>` 证明钥匙在、并指出用的是哪一行；**`ROOT LOGIN REFUSED`** 才是原因（`PermitRootLogin no` 的专属日志）；`refusing previously-used … key` 只是客户端重试所致，不是密钥损坏；`[preauth]` 表示**在认证完成前就被断开**。（另注：本机 `SyslogFacility AUTH` → sshd 自身日志在 `/var/log/messages`，不在 `/var/log/secure`。）

- **"手工配 `authorized_keys` 有什么隐藏坑？"** → **SELinux 标签**。手工 `cat >` 造的文件会**继承父目录标签**（`/home/xxx` 是 `user_home_t`），而 **sshd 只认 `ssh_home_t`** → 表现为"密钥配了却登不上"、**且不一定有显眼报错（静默失败）**。解法：**`restorecon -Rv <家目录>/.ssh`**（按策略打默认标签，比 `chcon` 持久），再用 **`ls -lZ`** 复核。同时权限"三件套"缺一不可：`.ssh` 700 + `authorized_keys` 600 + 家目录不可被组/他人写。
