# openEuler 练手笔记 · 2026-09-14（SSH 加固 + 包管理深度 + 用户权限）

> 机器：128（oe-base，openEuler 24.03 LTS，kernel 6.6.0-145.0.13.139.oe2403）
> 目标：① W1 加练「SSH 改端口」 ② DAY2 后半程「rpm / dnf 深度」 ③ DAY3 前半程「用户与权限体系」
> 风格：每条 = 命令 + 现象 + 为什么 + 类比（证据链）

---

## 一、SSH 加固：把 22 换成 2222（要过三关）

```bash
# ① 侦察
getenforce                                            # Enforcing
firewall-cmd --state                                  # running
ss -tlnp | grep :22                                   # sshd 听 0.0.0.0:22 + [::]:22
grep -inE "^#?port" /etc/ssh/sshd_config              # 21:#Port 22（注释态，走默认）

# ② 开新路（只增不减，零风险）
dnf install -y policycoreutils-python-utils           # 最小化安装默认没有 semanage
semanage port -a -t ssh_port_t -p tcp 2222            # 给 SELinux 放行 2222
firewall-cmd --permanent --add-port=2222/tcp && firewall-cmd --reload

# ③ 改配置
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak      # 改前备份（纪律）
sed -i 's/^#Port 22/Port 22/' /etc/ssh/sshd_config
echo "Port 2222" >> /etc/ssh/sshd_config
grep -nE "^Port" /etc/ssh/sshd_config                 # 两行：21:Port 22 / 161:Port 2222
sshd -t                                               # 语法自检，静默通过
systemctl restart sshd
ss -tlnp | grep sshd
```

**现象 / 结论**

| 检查点 | 回显 | 结论 |
|---|---|---|
| SELinux 放行 | `ssh_port_t tcp 2222, 22` | ✅ 新的端口进了安检白名单 |
| 防火墙放行 | `firewall-cmd --list-ports` → `2222/tcp` | ✅ 门卫登记了新门牌 |
| 双端口监听 | `ss -tlnp \| grep sshd` → **4 条 LISTEN**：`0.0.0.0:22`、`0.0.0.0:2222`、`[::]:22`、`[::]:2222`（同 pid） | ✅ 22 与 2222 并存 |
| 真机验证 | 新开 Xshell 会话连 `192.168.252.128:2222` → `Connection established` | ✅ 新端口真可用 |
| 回退通道 | 22 保留未撤 | ✅ 安全侧留退路 |

**坑位（必记）**

- **`Port` 是「叠加型」指令**：一旦配置文件里出现**任意一条** `Port`，默认的 22 **立即失效**。所以想"22 + 2222 同时听"**必须两条都写**；只写 `Port 2222` 会把 22 弄丢，而新端口还没验证过 → **把自己锁在门外**。
- **改 SSH 端口要过三关**：SELinux（`semanage port`）→ firewalld（`firewall-cmd --add-port`）→ sshd_config（`Port`）。**少任何一关都白改**（SELinux 是最容易被漏掉的那关）。
- **安全顺序不能乱**：先开新路 → `sshd -t` 通过 → restart → **新开会话验证成功** → 才考虑撤旧端口。

**类比**：改 SSH 端口像给大楼换大门 —— `semanage` 给新门办通行证，`firewall-cmd` 让门卫放行，`sshd_config` 改门牌号。顺序永远是"先开新门、验证能进，再撤旧门"。

---

## 二、rpm 四查（包 ↔ 文件，双向查询）

```bash
rpm -q nginx                     # nginx-1.24.0-10.oe2403.x86_64
rpm -qi nginx                    # 详情卡：版本/架构/安装时间/大小/许可/签名/编译机
rpm -ql nginx | head -20         # 这个包往哪些路径装了文件
rpm -qf /usr/sbin/nginx          # nginx-1.24.0-10.oe2403.x86_64（反查归属）
```

**三问一张表（背下来）**

| 想查什么 | 命令 | 类比 |
|---|---|---|
| 这个包装没装、什么版本 | `rpm -q nginx` | 查身份证在不在 |
| 这个包往哪铺文件（正查） | `rpm -ql nginx` | 查这人名下有几套房 |
| 这个文件属于哪个包（反查） | `rpm -qf /usr/sbin/nginx` | 拿到快递，扫码查"谁寄的" |
| 包的完整档案 | `rpm -qi nginx` | 调出完整人事档案 |

**从 `rpm -qi` 回显里读出的信息**

```
Name        : nginx
Epoch       : 1                              ← 版本比较的最高权重
Version     : 1.24.0
Release     : 10.oe2403
Install Date: Fri 11 Sep 2026 10:15:53 AM CST
Size        : 1421748
License     : BSD
Build Host  : node-k8gs7.compass-ci.net      ← openEuler 官方 CI 编译机
Packager    : http://openeuler.org           ← 官方源
```

- **`Epoch` 是版本比较的最高权重**：判断"哪个版本更新"的顺序是 **Epoch > Version > Release**。
- **`Build Host: ...compass-ci.net` + `Packager: openeuler.org`** → 确认包来自**官方源**，不是第三方塞进来的。
- **`Install Date` 是"这批文件最后一次被写下的时间"，不是最初安装时间** —— 见第四节的反例。

**这个指纹比对还能反查改动**：`rpm -V nginx` → **无输出 = 文件全部未被改动**（空回显就是最好的回显）。

**类比**：`rpm -qf` 像拿到一件快递扫码查寄件人；`rpm -V` 像回家调监控，看家具被动过没。空回显 = 一切照旧。

---

## 三、dnf 仓库视角（未装也能查）

```bash
dnf repoquery -l nginx | head -20     # 仓库视角：这包装完会铺哪些文件
dnf provides /usr/sbin/nginx          # 仓库视角：反查 + 列出全仓库候选版本
```

**`dnf provides` 回显（信息量最大的一段）**

```
nginx-1:1.24.0-1.oe2403.x86_64    Repo: everything     ← 基线仓库（出厂版）
nginx-1:1.24.0-2.oe2403.x86_64    Repo: update
nginx-1:1.24.0-3.oe2403.x86_64    Repo: update
…（5/6/7/8/9）                      Repo: update
nginx-1:1.24.0-10.oe2403.x86_64   Repo: @System        ← 本机已装的就是它
nginx-1:1.24.0-10.oe2403.x86_64   Repo: update
```

**三个 Repo 的含义（必须记住）**

| Repo | 含义 |
|---|---|
| `everything` | 发行版**初始仓库**（LTS 基线） |
| `update` | **更新仓库**，后续补丁都发这里 |
| `@System` | **本机已安装**的包（在系统里，不在仓库里） |

**读出的三条结论**

1. 本机装的 `-10` 正是 update 仓库里**最新的** → **无需升级** ✅
2. nginx 从 `-1` 一路打到 `-10` → 这包**更新很勤**（安全补丁多）
3. `rpm -qf` 只能看到 `@System` 一条；`dnf provides` 能看到**全仓库候选** → 这就是"升级前先看看有哪些版本"的标准姿势

**两对"视角"对照表**

| 想查什么 | 已装视角 | 未装/仓库视角 |
|---|---|---|
| 这个包装了哪些文件 | `rpm -ql nginx` | `dnf repoquery -l nginx` 或 `rpm -qpl xxx.rpm` |
| 这个文件属于哪个包 | `rpm -qf /usr/sbin/nginx` | `dnf provides /usr/sbin/nginx` |

**顺带一个 dnf 机制**

```bash
# 每条 dnf 命令第一行会打印：
Last metadata expiration check: 1:21:33 ago on Mon 14 Sep 2026 08:31:15 AM CST.
```

含义 = "上次检查仓库元数据是 1 小时 21 分前"。dnf 把元数据缓存在 `/var/cache/dnf/`，默认 `metadata_expire=48h` 内**直接用缓存、不联网** —— 这是 dnf 比 yum 快的机制之一。强制刷新：`dnf clean metadata`。

---

## 四、离线下载与事务审计（运维实战）

```bash
mkdir -p /tmp/rpmtest && dnf download --resolve nginx --destdir=/tmp/rpmtest
dnf history | head -20
```

### 4.1 `--resolve` 的陷阱：只下"本机缺的"依赖

回显**只下了 1 个包**（`nginx-1.24.0-10.oe2403.x86_64.rpm`，498 kB）。但 nginx 明明有十几个依赖（9/11 装它时 Altered = 14 个包）—— 为什么？

因为 **`--resolve` 的语义是"补齐本机缺失的依赖"**。本机 openssl-libs / pcre2 / zlib / nginx-mod-* 都装好了，它认为"不缺"，一个都不下。

| 参数 | 含义 | 结果 |
|---|---|---|
| `--resolve` | 只下系统里**缺的**依赖 | 只 1 个包 |
| `--resolve --alldeps` | 已装的**也下**（不跳过） | 才能真离线装机 |

**离线装机标准动作**：外网机 `dnf download --resolve --alldeps` 拉全套 → 拷进内网 → `dnf install *.rpm`。

**类比**：`--resolve` 像"出门前查缺什么补什么"；`--alldeps` 像"不管家里有没有，整套行李都装箱" —— 因为目的地（内网新机器）**什么都没有**。

### 4.2 `dnf history`：这台机的"生命史"

```
ID | Command line                              | Date and time     | Action(s) | Altered
 8 | install -y policycoreutils-python-utils   | 2026-09-14 09:24  | Install   |     8
 7 | history undo 6                            | 2026-09-11 10:15  | Install   |    14
 6 | remove -y nginx                           | 2026-09-11 10:15  | Removed   |    14
 5 | install -y nginx                          | 2026-09-11 10:14  | Install   |    14
 4 | install -y vim wget curl net-tools …      | 2026-09-10 17:58  | Install   |     9
 3 | update -y                                 | 2026-09-10 17:43  | I, U      |   205 EE
 2 | install -y vim net-tools                  | 2026-09-10 09:45  | Install   |     5 <
 1 | （空）                                     | 2026-09-02 22:10  | Install   |   521 >E
```

**`Action(s)` 缩写**：`I`=Install　`U`=Upgrade　`D`=Downgrade　`E`=Remove　`O`=Obsolete　`R`=Reinstall　`C`=Reason change

**`Altered` 尾部符号（官方文档）**

| 符号 | 含义 |
|---|---|
| `>` | 事务**之后**，RPM 数据库被 DNF 之外的东西改过 |
| `<` | 事务**之前**被改过 |
| `*` | 事务**未完成就中止** |
| `#` | 完成但**退出码非 0** |
| `E` | 成功但**有警告/报错输出** |

> ⚠️ 别混：`E` 在 **Action(s)** 列是"删除"，在 **Altered** 列才是"有警告"。

**常用延伸**

```bash
dnf history info 8         # 看某事务完整明细（命令行不截断）
dnf history undo 8         # 反向回滚这个事务
dnf history rollback 5     # 回滚到 ID 5 之后的所有事务
```

### 4.3 三处交叉验证（这才是重点）

| history 记录 | 另一条独立证据 | 结论 |
|---|---|---|
| `ID 8` 09-14 **09:24**，**8 个**包 | `rpm -qa --last` 里 09:24:28 那批**也是 8 个** | ✅ 互证 |
| `ID 5/6/7` 9-11 **10:14 装 → 10:15 删 → 10:15 undo** | `rpm -qi nginx` 的 `Install Date = 2026-09-11 10:15:53` | ✅ **现存 nginx 是 ID 7（undo）装回来的，不是 ID 5** |
| `ID 3` 09-10 17:43 `update -y`，**205 个**包 | `/etc/cron.daily` 目录 mtime = `Sep 10 17:43` | ✅ 权限变更就是那次全量更新带来的 |

**方法论沉淀**：**排障闭环 = 拿证据 → 对时间线 → 查机器当时在干什么**

- `rpm -qf` / `rpm -V` → 定归属、定"变没变"
- `stat` / `ls -l` → 取时间戳
- `dnf history` → 机器当时在干什么

三样一叠，事件就能还原。**别停在"报错了"这三个字上。**

### 4.4 `rpm -Va`：全量校验怎么读

```bash
rpm -Va 2>&1 | head -20
rpm -Va 2>&1 | grep -vE ' (c|d|g|l|r) '      # ★ 过滤噪声，只留真信号
```

**第一列：9 位标记（顺序固定）**

| 位置 | 字母 | 含义 |
|---|---|---|
| 1 | `S` | 大小变 |
| 2 | `M` | 权限/模式变 |
| 3 | `5` | 摘要（MD5/SHA）变 → **内容被改** |
| 4 | `D` | 设备号变 |
| 5 | `L` | 符号链接目标变 |
| 6 | `U` | 属主变 |
| 7 | `G` | 属组变 |
| 8 | `T` | mtime 变 |
| 9 | `P` | capabilities 变 |

`.` = 该位没变。`S.5....T.` = 大小+摘要+时间都变；`M.......` = 只有权限变。

**第三列：属性位（噪声来源）**

| 属性 | 含义 | 噪声等级 |
|---|---|---|
| `c` | 配置文件 | 高：被改是常态 |
| `d` / `l` / `r` | 文档 / 许可 / 说明 | 高 |
| `g` | **ghost**：包不装它，运行时生成 | 高：几乎必然有差异 |

行首 **`missing`** = rpm 库里有记录、磁盘上文件没了。

**本地实例（128）判定**

| 行 | 判定 |
|---|---|
| `S.5....T.  c /etc/issue` 等 | ✅ 配置文件被改，**常态** |
| `M.......  g /var/spool/anacron/*`、`g /var/lib/logrotate/logrotate.status` | ✅ **ghost 运行时文件**，差异完全正常 |
| `M.......  /etc/cron.daily` 等 cron 目录 | ✅ 实际 700 vs 库里 755，**RHEL 系老牌假阳性** |
| `missing  c /etc/cron.deny` | ⚠️ 需追问"谁删的" |
| `S.5....T.  /usr/lib/udev/rules.d/50-udev-default.rules` | 🔴 **真信号**：非配置文件被改（归 `systemd-udev` 所有） |

**过滤原理**：小写 `c d g l r` **只会出现在属性位**（标记位全是大写或 `.`），剔掉它们，剩下的才是**真正的系统文件差异**。

**类比**：`rpm -Va` 像全楼消防大检查 —— 一定报一堆"灯罩松了、贴纸歪了"，你的本事是**从噪音里挑出那扇被撬开的门**。

---

## 五、用户与组（DAY3）

```bash
useradd -m -s /bin/bash devops
echo 'Devops@123' | passwd --stdin devops
id devops                        # uid=1000(devops) gid=1000(devops) groups=1000(devops)
grep '^devops' /etc/passwd       # devops:x:1000:1000::/home/devops:/bin/bash
passwd -S devops                 # devops PS 2026-09-14 0 99999 7 -1 (Password set, SHA512 crypt.)
ls -l /etc/shadow                # ----------. 1 root root 763 Sep 14 12:07 /etc/shadow
```

**为什么是 1000**：RHEL 系普通用户从 **1000** 起（0 = root，1~999 留给系统账号）。

**`/etc/passwd` 7 段结构**

```
devops : x : 1000 : 1000 :  : /home/devops : /bin/bash
用户名  密码  UID    GID   描述  家目录       登录shell
```

> 那个 **`x` 是关键**：密码哈希**不在这**，只是个占位符 —— 哈希真正的家在 `/etc/shadow`。

**`/etc/shadow` 权限 `----------`（000）** 只有 root 能读。**这正是不把哈希放 `/etc/passwd` 的原因**（后者全世界可读）。行尾那个 `.` = 有 SELinux 安全上下文。

**`passwd -S` 字段解码**

| 字段 | 值 | 含义 |
|---|---|---|
| 状态 | `PS` | Password Set（`L`=锁定，`NP`=无密码） |
| 最后修改 | `2026-09-14` | 上次改密码日期 |
| 最小天数 | `0` | 改完可立即再改 |
| **最大天数** | **`99999`** | ≈ **永不过期** ⚠️ 等保要收紧 |
| 警告天数 | `7` | 到期前 7 天提醒 |
| 失效天数 | `-1` | 过期后账号不失效 |
| 算法 | `SHA512 crypt` | 哈希算法 |

**收紧密码策略（等保 2.0 常要求 90 天）**

```bash
chage -M 90 devops && chage -l devops
# Last password change : Sep 14, 2026
# Password expires     : Dec 13, 2026      ← 90 天后
# Maximum number of days between password change : 90
```

**组与附加组（这里有最大的坑）**

```bash
groupadd -g 3000 appgrp
usermod -aG appgrp devops
id devops                 # groups=1000(devops),3000(appgrp)
getent group appgrp       # appgrp:x:3000:devops
```

**⚠️ `usermod -aG` 的 `-a` 是保命符**

```bash
usermod -aG wheel devops && id devops
# groups=1000(devops),10(wheel),3000(appgrp)      ← wheel 在

usermod -G appgrp devops && id devops
# groups=1000(devops),3000(appgrp)                ← wheel 没了！
```

**漏掉 `-a`，会把该用户原有的附加组列表整张覆盖掉。** 生产上"某人突然没了 sudo 权限""突然进不了 Docker 组"，十有八九就是这么来的。

**类比**：`-aG` 像"往名单末尾**追加**一行"；`-G` 像"把整张名单**撕了重写**"。

---

## 六、权限特殊位：SUID / SGID / Sticky

### 6.1 SGID（目录的"组继承"）

```bash
mkdir -p /srv/share && chgrp appgrp /srv/share && chmod 2775 /srv/share
ls -ld /srv/share                       # drwxrwsr-x. 2 root appgrp 4096 …   ← 组位 s = SGID
su - devops -c 'touch /srv/share/a.txt'
ls -l /srv/share/a.txt                  # -rw-r--r--. 1 devops appgrp 0 …     ← 属组是 appgrp！
```

- `chmod 2775` 的**首位 `2` 就是 SGID**。完整权限是 **4 位数字**：`2`(SGID) `7`(属主) `7`(属组) `5`(其他)。
- 文件属主是 `devops`（谁建的归谁），但**属组自动变成 `appgrp`** —— SGID 在起作用。
- **业务场景**：多人协作目录。没有 SGID 时每人建的文件属组是自己的私有组，别人读不了；有 SGID 属组统一，配 `chmod g+w` 就**全组可读写**。

### 6.2 Sticky（"只能删自己的"）

```bash
ls -ld /tmp        # drwxrwxrwt. 12 root root 240 …     ← 末位 t = Sticky
```

- `chmod 1777` 的**首位 `1` 是 Sticky**；`/tmp` 出厂就是 `1777`。
- 含义：目录里**任何人都能建文件**，但**只有文件属主（或 root / 目录属主）能删自己的**。
- 没有它，`/tmp` 里任何人都能删别人的文件 —— 灾难。

### 6.3 SUID 基线清单（背下来）

```bash
find /usr/bin -perm -4000 -type f
```

本机（128）**14 个 SUID 程序 = 基线**：

```
/usr/bin/chage        /usr/bin/crontab     /usr/bin/fusermount   /usr/bin/newgidmap
/usr/bin/gpasswd      /usr/bin/passwd      /usr/bin/pkexec       /usr/bin/su
/usr/bin/umount       /usr/bin/newuidmap   /usr/bin/mount        /usr/bin/newgrp
/usr/bin/sudo         /usr/bin/staprun
```

**为什么必须 SUID**：它们都要干"当前用户权限不够"的事。

| 程序 | 为什么需要 SUID |
|---|---|
| `passwd` | 普通用户改自己密码，但要写 root 独占的 `/etc/shadow` |
| `su` / `sudo` / `pkexec` | 提权 |
| `mount` / `umount` / `fusermount` | 挂载 |
| `chage` / `gpasswd` / `crontab` / `newgrp` | 账户与计划任务 |
| `newuidmap` / `newgidmap` | 用户命名空间（**容器的基础**） |
| `staprun` | systemtap 内核探针 |

**⚠️ 为什么这份清单是高危面**：SUID 的本质是"**以 root 身份运行你的输入**"，一有漏洞就是本地提权。

| 案例 | 要点 |
|---|---|
| **`pkexec` / CVE-2021-4034（PwnKit）** | 参数处理缺陷 → 任意用户提权 root，2022 年最响的洞 |
| **`staprun`** | 能读内核内存 → 等保/加固基线常要求摘 SUID 位：`chmod u-s /usr/bin/staprun` |

**建基线 + 定期比对（把它变资产）**

```bash
find / -perm -4000 -type f 2>/dev/null | sort > /root/suid-baseline.txt
diff <(find / -perm -4000 -type f 2>/dev/null | sort) /root/suid-baseline.txt
```

> **多出一个不认识的 SUID 程序 = 入侵信号** —— 攻击者拿到低权 shell 后常自己塞一个做持久化提权。

---

## 七、今日踩坑清单

| 踩坑 | 原因 | 正确做法 |
|---|---|---|
| `rpm -qi nginx` 打出用法帮助页，重打一次就好 | **输入层**：中文输入法导致 `-` / 字母变形 | 敲命令前切英文输入法，选项用 `Tab` 补全 |
| `dnf repoquory` / `hrad` | 同上，手打拼错 | 同上 |
| `ls -lh` → `cannot access 'lh'` | 同上，`-` 没被识别成选项前缀 | 同上；**报错同时正常参数照样执行 = "部分成功"，脚本里要 `set -e`** |
| 以为 `usermod -G` 和 `-aG` 等价 | 漏 `-a` 会**覆盖**全部附加组 | 永远写 `usermod -aG` |
| 以为 `dnf download --resolve` 会下全套依赖 | 它**只下本机缺的** | 离线装机要加 `--alldeps` |
| 以为 `rpm -qi` 的 Install Date 是最初安装时间 | 它是**最后一次落盘时间** | 要还原来龙去脉，配 `dnf history` |
| 把 `rpm -Va` 的 `c` / `g` 行当真故障 | 配置与 ghost 文件差异是常态 | 先 `grep -vE ' (c\|d\|g\|l\|r) '` 过滤 |

**工具箱通用铁律**

> 工具**打 usage 而不打业务报错** = **参数解析失败**。排查优先级：
> ① 输入层（全角/粘贴不可见字符）→ ② 拼错 → ③ 短选项不被认（换长选项）→ ④ 被 `alias` 函数劫持（`type <命令>`）
>
> 另一条元教训：**单次现象不足以定因，要用"再跑一次"收口。**

---

## 八、面试一句话（直接当素材）

- **"SSH 怎么改端口？"** → 要过三关：SELinux（`semanage port -a -t ssh_port_t`）、防火墙（`firewall-cmd --add-port`）、sshd 配置（`Port`）；且 `Port` 是叠加型 —— 写了任意一条默认 22 就失效，想并存必须两条都写。正确顺序是**先开新端口并验证连通、再撤旧端口**，避免把自己锁在门外。

- **"服务器上一个陌生文件，怎么知道是谁装的？"** → 已装的用 `rpm -qf <路径>`，仓库里的用 `dnf provides <路径>`；反过来查一个包装了哪些文件，已装的 `rpm -ql`、未装的 `dnf repoquery -l` 或 `rpm -qpl xxx.rpm`。

- **"怀疑系统文件被篡改怎么查？"** → `rpm -Va` 做全量校验，9 位标记里 `5` 表示摘要变化（内容被改）；但输出里 `c`（配置文件）和 `g`（运行时 ghost 文件）的差异是常态，要先 `grep -vE ' (c|d|g|l|r) '` 过滤，真正的红旗是系统二进制出现 `5` 或 `missing`。

- **"怎么查一台机器最近被动过什么？"** → `dnf history` 给事务级日志（命令行、时间、动作、影响包数），`rpm -qi` 给文件级时间戳，两者一组合能还原完整变更链路（含"装过、删过、又 undo 回来"）；注意 `Install Date` 是**最后一次落盘时间**，不是首发时间。

- **"离线/内网怎么装包？"** → 外网机 `dnf download --resolve --alldeps` 把包和**全部依赖**下齐（只写 `--resolve` 只会下本机缺的），拷进内网后 `dnf install *.rpm` 或搭本地 repo。

- **"用户权限怎么管？"** → 建用户 `useradd -m -s /bin/bash`（`-m` 建家目录最容易漏）；密码哈希存 `/etc/shadow`（权限 000、root 独占），`/etc/passwd` 里只是 `x` 占位；`passwd -S` 看密码策略，默认 `99999` 永不过期，等保用 `chage -M 90` 收紧；加附加组**必须** `usermod -aG`，漏 `-a` 会覆盖原有全部附加组。

- **"SUID 有什么风险？"** → SUID 让程序以属主身份运行（本质是"以 root 运行用户输入"），是提权必需机制也是最大攻击面；用 `find / -perm -4000` 建基线并定期比对，多出陌生 SUID 程序就是入侵信号；典型案例如 pkexec 的 PwnKit（CVE-2021-4034），另外等保基线常要求摘掉 `staprun` 的 SUID 位。

- **"SGID 和 Sticky 的区别？"** → SGID（`2775`）用在**目录**上让新建文件继承目录属组，适合多人协作目录；Sticky（`1777`）让目录里只能删自己的文件，`/tmp` 是样板。
