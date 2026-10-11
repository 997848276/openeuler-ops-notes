# day31 · 用户、权限与 ACL 精讲：从家底侦察到 /root 550 悬案（2026-10-10 · 128 oe-base）

## 一 侦察：家底全景

```bash
awk -F: '$3>=1000 || $3==0 {printf "%-10s UID=%-5s\n",$1,$3}' /etc/passwd
# root(0) / nobody(65534) / devops(1000) / netops(1001)
grep -E '^UID_MIN|^UMASK' /etc/login.defs      # UID_MIN 1000 / UMASK 077
id devops                                       # devops wheel appgrp(3000) isula(989)
find /usr/bin /usr/sbin -perm /6000 -type f    # passwd/su/sudo/chage/crontab...
ls -ld /tmp /var/tmp                           # drwxrwxrwt ← sticky 位
mount | grep -E ' on / | on /data '            # /=ext4 /data=xfs(data_vg-data_lv)，都带 seclabel
sudo -l                                        # (ALL) NOPASSWD: ALL；Defaults always_set_home 实锤
```

- UID 1000 分界线；xfs 原生 ACL；sudo -l 是提权面审计标准动作。

## 二 umask 双源案：077 和 022 都是真的

```bash
umask                                          # 0022
grep -E '^UMASK' /etc/login.defs               # 077
grep -n 'umask' /etc/profile /etc/bashrc       # bashrc:70 只在 umask==0 时兜底 022
grep -rn 'pam_umask' /etc/pam.d/               # 无 → 077 不经 PAM
sudo ls -ld /home/devops /home/netops          # drwx------ (700) ← 077 的出生证明
```

- ★login.defs 的 077 只归 useradd 管；日常 0022 从 sshd 继承 + bashrc 兜底。
- ★坑：配置文件里的数 ≠ 当前 shell 的数，先问"这数归谁管"。

## 三 setuid 与 sticky 活体

```bash
ls -l /usr/bin/passwd /etc/shadow
# -rwsr-xr-x  passwd（owner x→s 临时工牌） / ----------  shadow（0000）
touch /tmp/sticky-devops.txt
sudo -u netops rm -f /tmp/sticky-devops.txt    # rc=1 Operation not permitted（储物柜规则）
rm -f /tmp/sticky-devops.txt                   # rc=0
```

## 四 setgid 共享目录 + 今天最大坑

```bash
sudo groupadd proj1 && sudo usermod -aG proj1 devops && sudo usermod -aG proj1 netops
sudo mkdir -p /data/proj1 && sudo chown root:proj1 /data/proj1 && sudo chmod 2770 /data/proj1
ls -ld /data/proj1                             # drwxrws--- ← 组 x→s
touch /data/proj1/a.txt                        # rc=1 Permission denied ★★
sg proj1 -c 'touch /data/proj1/a.txt && ls -l /data/proj1/a.txt'
# -rw-r--r-- 1 devops proj1 ...（组=proj1，setgid 继承）
sudo -u netops touch /data/proj1/b.txt         # sudo -u 新进程副组即时生效
sudo -u netops rm /data/proj1/a.txt            # rm 提示 write-protected? 敲 y → 删掉
```

- ★★usermod -aG 后旧会话凭证不热更新：正解 `sg <组> -c` 或重登（touch/ls 双被拒同一坑）。
- ★rm 的 write-protected 提示不是拒绝：删除裁决者=目录 w 位。
- ★2770 无 sticky → 同组互删（对照 /tmp sticky）。

## 五 /root 550 悬案：排障方法论全程实弹

```bash
sudo getfacl /root            # user::r-x group::r-x other::---
sudo stat -c 'mode=%A (%a)' /root    # mode=dr-xr-x--- (550) ★owner 竟无 w
ls /root; echo rc=$?          # Permission denied rc=2（纯 DAC）
id -Z                         # unconfined_u:unconfined_r:unconfined_t
sudo ausearch -m avc -ts today | grep -i denied   # 只有 rngd/agetty 老噪音
sudo chmod 700 /root          # 复原 drwx------ (700)
```

- ★★★三证合一：/root=550，devops 被拒=纯 DAC（other=0），SELinux 无辜（unconfined_t+无新 avc+模式自洽）。
- ★取证三坑：①mtime 是红鲱鱼（文件增删刷它，chmod 只改 ctime）；②热闹目录 ctime 也会被内容变更覆盖；③ausearch -ts recent 只有 10 分钟窗口（窗口选错=假阴性）。
- ★550（owner 无 w）绝非发行版默认，嫌疑=9 月加固照教程 chmod；修复=700。
- ★ls x 2>/dev/null 静默不可作判据，判定带 rc。

## 六 ACL：点名授权 + mask 天花板 + default 继承

```bash
sg proj1 -c 'echo test >> /data/proj1/b.txt'          # rc=1（644 组只读）
sudo setfacl -m g:proj1:rw /data/proj1/b.txt
sudo getfacl /data/proj1/b.txt
# user::rw- / group::r-- / group:proj1:rw- / mask::rw- / other::r--
ls -l /data/proj1/b.txt                               # -rw-rw-r--+ ← + 号=带 ACL
sg proj1 -c 'echo ok >> /data/proj1/b.txt'            # rc=0

sudo setfacl -m mask::r /data/proj1/b.txt             # 天花板钳制
sudo getfacl /data/proj1/b.txt | grep proj1           # group:proj1:rw- #effective:r--
sg proj1 -c 'echo no >> /data/proj1/b.txt'            # rc=1 → mask::rw 恢复

sudo setfacl -d -m g:proj1:rw /data/proj1             # default：出生规则
sg proj1 -c 'touch /data/proj1/c.txt && getfacl /data/proj1/c.txt | grep -E "proj1|mask"'
# group:proj1:rw- / mask::rw- ← 出生即组写
sg proj1 -c 'echo d1 >> /data/proj1/c.txt'            # rc=0
sudo -u netops sh -c 'echo d2 >> /data/proj1/c.txt'   # rc=0 ← 双向组写

# 清场
sudo rm -rf /data/proj1; sudo gpasswd -d devops proj1; sudo gpasswd -d netops proj1
sudo groupdel proj1; getent group proj1               # 空 rc=2
```

- ★ACL=点名名单（不占 u/g/o 正编），mask=名单天花板（#effective 即钳制后真实权限）。
- ★部门协作目录三件套：setgid 保组（2770）+ default ACL 保组写 + 知道无 sticky 会互删。

## 七 坑位速查

| 坑 | 正解 |
|---|---|
| usermod -aG 不生效 | sg <组> -c 或重登（旧会话凭证不热更新） |
| UMASK 077 vs umask 022 | 077 归 useradd；022 从 sshd 继承+bashrc 兜底 |
| 目录 mtime 变=被 chmod？ | 红鲱鱼；chmod 改 ctime；热闹目录 ctime 也被覆盖 |
| ausearch recent 查无 | 窗口仅 10 分钟；用 -ts today |
| 2>/dev/null 静默 | 不可作判据，判定带 rc |
| 组友写不了 setfacl 后的文件 | 看 mask #effective；mask::rw 恢复 |
| 新文件组友写不了 | 目录 default ACL：setfacl -d -m g:组:rw |
| 判断 DAC 还是 SELinux 拒 | 模式位+id -Z+ausearch 三证合一 |

## 面试一句话

> "Linux 权限我按四层讲：权限位+umask（077 归 useradd、日常 022 从 sshd 继承，配置里的数不等于 shell 的数）；特殊位（setuid 临时工牌——passwd 靠它写 0000 的 shadow；sticky 储物柜；setgid 目录自动继承组，但 usermod -aG 后旧会话不热更新，必须 sg 或重登——这是我实测踩过的坑）；ACL（点名授权不占正编、mask 是天花板、default ACL 让新文件出生即组写）；方法论——我排查过 /root 权限悬案：记忆 700、实测 550，先疑 SELinux，但 unconfined_t+全天无 avc+模式位自洽，三证合一定性纯 DAC；取证途中踩了 mtime 红鲱鱼和 ausearch 10 分钟窗口两个坑。判断被谁拒绝，永远模式位、SELinux 域、audit 记录三证合一。"
