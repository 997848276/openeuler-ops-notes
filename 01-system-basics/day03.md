# Day 03 · 系统环境校准与 SSH 安全加固

日期：2026-09-12 ｜ 环境：openEuler 24.03 LTS (oe-base 192.168.252.128)

## 一、环境体检

| 检查项 | 命令 | 结果 |
|---|---|---|
| 主机名 | `hostnamectl` | oe-base |
| 时区 | `timedatectl` | Asia/Shanghai (CST, +0800)，NTP active |
| 字符集 | `locale` | LANG=en_US.UTF-8 |
| 中文语言包 | `localectl list-locales \| grep -i zh` | 4 个（zh_CN/zh_HK/zh_SG/zh_TW）|

> 观察：`RTC in local TZ: no` 是 Linux 标准做法（硬件时钟存 UTC），
> 只有与 Windows 双系统共机时才需要改。

## 二、实战排障：主机名解析慢 5 秒

**现象**：`ping -c 2 oe-base` 耗时 5209ms（丢包 0%、延时 0.046ms）

**排查过程**：
1. `getent hosts oe-base` → 解析成 IPv6 链路本地地址 fe80::...
2. `grep -n "oe-base" /etc/hosts /etc/hostname` → **/etc/hosts 里没有映射**
3. `cat /etc/resolv.conf` → nameserver 114.114.114.114 + **8.8.8.8（国内不可达）**&#8203;

**根因**：/etc/hosts 缺少主机名映射 → 解析绕过本地表 → 去问 DNS →
8.8.8.8 超时拖慢 5 秒

**处理**：
· echo "192.168.252.128  oe-base" >> /etc/hosts
· sed -i.bak '/8.8.8.8/d' /etc/resolv.conf

**验证**：`time getent hosts oe-base` → real 0m0.035s（**5.2s → 0.035s，提速约 150 倍**）

**踩坑记录**：Xshell 粘贴吞掉单引号 → sed 报
`-e expression #1, char 2: unknown command: '.'`
解决：粘完先肉眼检查引号；或改用无引号写法 `sed -i.bak /8.8.8.8/d 文件`

## 三、SSH 安全加固（先开后关）

### 加固前状态
· permitrootlogin yes
· passwordauthentication yes
· ✅ 有 id_ed25519（但要确认 authorized_keys 是否存在！）

### 步骤 1：建立密钥通道（先开）
1. Windows 本地生成：`ssh-keygen -t ed25519 -C "leihao-win"`
2. 服务器写入公钥：`echo "ssh-ed25519 AAAA... leihao-win" >> ~/.ssh/authorized_keys`
3. 权限（**错了 sshd 会静默拒绝**）：
   · chmod 700 ~/.ssh
   · chmod 600 ~/.ssh/authorized_keys
4. **验证密钥可登录**（新窗口 `ssh root@IP`，不问密码 → 成功）

### 步骤 2：关闭密码通道（后关）
· sed -i.bak 's/^#\?PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
· sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
· sshd -t          ← **语法检查，必须做**
· systemctl reload sshd    ← **用 reload 不用 restart**

### 步骤 3：双向验证
· 密钥登录 → ✅ 成功
· 强制密码登录 → ✅ `Permission denied (publickey,gssapi-keyex,gssapi-with-mic)`
  · **注意：括号里 password 消失了 = 加固生效**

### 知识点
· `prohibit-password` 在 RHEL/openEuler 显示为 **`without-password`**，两者等价
· `reload` = SIGHUP，主进程不换、已有连接不断（日志可见同 PID 1063）
· `restart` = 杀进程重启，有断连风险
· 为什么 chmod 700/600：权限过开放时 sshd 拒绝使用密钥（防他人篡改）

## 四、日志分析

· `journalctl -u sshd --since "10 minutes ago"` → 看到 Invalid user / Accepted publickey
· `journalctl -u sshd -f` → 实时跟踪（Ctrl+C 退出）
· `last -5` → 五个会话，每个都能对应今天的操作
· `/var/log/wtmp` 最早记录 Sep 2 22:15:18

**密钥认证三阶段（日志可证）**&#8203;：
1. Accepted key ... found at /root/.ssh/authorized_keys:1:1   （找到公钥）
2. Postponed publickey ...                                    （待验证私钥）
3. Accepted publickey ... ED25519 SHA256:Fm3Pd...             （签名通过）

**观察到的噪声**：`Invalid user ssh root` —— Xshell 登录名栏误填 "ssh root"，
把命令当用户名。真实环境中大量此类记录来自公网扫描。

## 五、本周加练项进度

| 加练项 | 状态 |
|---|---|
| sed -i.bak 就地替换 | ✅ 三次实战（resolv.conf / sshd_config）|
| sshd_config 加固 | ✅ 含双向验证 |
| journalctl 看服务日志 | ✅ journalctl -u / -f / last |

