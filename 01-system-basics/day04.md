# openEuler 练手笔记 · 2026-09-13（域二/三/四/五/六 补强）

> 机器：128（oe-base，openEuler 24.03）
> 目标：补强 24 项 Linux 基础自检里最弱的 5 个域（文本三剑客、服务日志、权限、磁盘、进程资源）
> 风格：每条 = 命令 + 现象 + 为什么 + 类比（证据链）

---

## 一、服务与日志巡检（域四 + 域二）

```bash
systemctl status sshd
journalctl -u sshd --since "2026-09-12" | grep -iE "accepted|failed" | awk '{print $1,$2,$3,$NF}'
journalctl -u sshd --since "2026-09-12" | awk '/Connection from/ {print $8}' | sort | uniq -c | sort -rn
journalctl -u sshd --since "2026-09-12" | grep -c "Accepted"   # = 3
journalctl -u sshd --since "2026-09-12" | grep -c "Failed"      # = 0
```

**现象 / 结论**
- `systemctl status sshd` → `enabled` + `active (running)`，主 PID 1087，说明服务活着且开机自启。
- `grep -c "Failed" = 0` → **0 次失败登录**，SSH 密钥加固生效。
- IP 排行输出 `1 192.168.252.1` → 只有 Windows 宿主（VMnet8）连过 sshd，且只连 1 次。

**坑位**
- sshd "Connection from" 行里，源 IP 在 **$8**，不是 $6（$6 是单词 `Connection`）。
- `NR==1` 只取 journalctl 第一条（`Starting OpenSSH server daemon`），字段结构和 `Connection from` 行不同，字段号不能跨行复用。
- 取字段前先用 `awk 'NR==1 {for(i=1;i<=NF;i++) print i":"$i}'` 看清每列序号。

**类比**：`systemctl status` 像床头监护仪显示服务生命体征；`journalctl` 像翻病历本；`grep -c` 像计数器；IP 排行像统计谁最常来串门。

---

## 二、权限与打包（域五 + 域一）

```bash
umask                                   # 0022
touch /tmp/perm-demo.txt && ls -l /tmp/perm-demo.txt   # -rw-r--r-- (644)
mkdir -p /tmp/perm-demo && chmod 750 /tmp/perm-demo && ls -ld /tmp/perm-demo  # drwxr-x--- (750)

mkdir -p /tmp/bk && echo hello > /tmp/bk/a.txt && echo cfg > /tmp/bk/b.conf
tar -czf /tmp/bk.tar.gz -C /tmp bk      # 加 -C，归档路径干净为 bk/
tar -tzf /tmp/bk.tar.gz                 # bk/  bk/a.txt  bk/b.conf
find /tmp/bk -name "*.conf"             # /tmp/bk/b.conf
```

**现象 / 结论**
- `umask 0022`：新建文件 666-022=**644**（`rw-r--r--`），目录 777-022=**755**。
- `chmod 750` → `rwxr-x---`：属主全权、属组读+执行、其他人无权限。
- **目录的 `x` = 能 `cd` 进去**；没有 `x` 连门都推不开，哪怕有 `r` 也列不出内容。

**坑位（必记）**
- `tar -C /tmp bk` 先 cd 到 /tmp 再打包 → 包里路径是 `bk/...`（干净）。
- 省掉 `-C` 直接 `tar -czf bk.tar.gz /tmp/bk` → 包里带绝对路径 `/tmp/bk`，解包时**强制还原覆盖原目录**。
- 口诀：**想让包里路径干净，先 `-C` 过去再打包。**

**类比**：umask 像装修时默认拆掉的隔断；750 像项目文件夹（你是负责人 rwx，同事 r-x，外人 ---）；`tar -C` 像先走进房间再装箱。

---

## 三、进程 / 资源 / 端口（域三 + 域六）

```bash
ps aux --sort=-%cpu | head -10         # 前 10 全是 [kworker/...] 内核线程，%CPU/%MEM 均 0.0
free -h                                 # Mem total≈3.3Gi used 560Mi available 2.7Gi；Swap 4Gi used 0B
df -h                                   # / 6%  /boot 29%  /home 1%
ss -tunlp                               # sshd:22  nginx:80(3 worker)  chronyd UDP 323
lsof -i :22                             # sshd PID 1061 在 IPv4/IPv6 *:ssh 上 LISTEN
```

**现象 / 结论（系统健康基线）**
- `ps` 前 10 全是 `[ ]` 内核线程、`0.0%` → 系统空载，正常。
- `free -h`：`available` 2.7Gi 远大于 `used` 560Mi，`Swap used=0B` → 无内存压力。
- `df -h`：根分区仅 6%，`/boot` 29%（未到 90% 告警线，巡检时留意即可）。
- `ss -tunlp`：22/80 正常监听；nginx 显示 3 个 worker PID（1105/1106/1107），是 "master + 多 worker" 模型。
- `lsof -i :22`：sshd 同时监听 IPv4 与 IPv6，两条都是 LISTEN 等待状态。

**类比**：`ps` 像车间巡视看谁最忙；`free` 看仓库空位；`df` 看储物柜塞多满；`ss` 看大楼门口岗位表；nginx 三个 worker 像三个售货员共用一个柜台。

---

## 四、今日踩坑清单

| 踩坑 | 正确做法 |
|---|---|
| awk 取错字段（$6 是 Connection） | 先 `awk 'NR==1 {for...}'` 看列号，IP 在 $8 |
| tar 没加 `-C` 导致包里带绝对路径 | 先 `-C <目录>` 再打包，路径才干净 |
| 只看 `grep -c "Accepted"` 当登录次数 | 更准确应数 `Accepted publickey` |
| 把 `[kworker]` 当异常进程 | `[ ]` 是内核线程，空载 0.0% 正常 |

---

## 五、面试一句话（直接当素材）

- **"服务器卡了怎么查？"** → 先 `ps aux --sort=-%cpu | head` 定位高 CPU 进程，再 `free -h` 看内存、`df -h` 看磁盘、`ss -tunlp` 看端口进程对应，最后 `lsof -i :端口` 锁定连接。
- **"怎么看 SSH 有没有被暴力破解？"** → `journalctl -u sshd | grep -c Failed`，配合 IP 排行定位来源；看到 0 说明密钥策略生效。
- **"tar 打包怎么避免覆盖原目录？"** → 用 `tar -czf 包名.tar.gz -C 目标目录 子目录`，`-C` 让归档路径干净。

---

## 六、下一步

- [ ] Xftp 传到 128 `~/openeuler-ops-notes/` 对应目录
- [ ] `git add . && git commit -m "openEuler 基础补强 9/13：服务日志+权限+进程端口" && git push`
- [ ] 把 A/B/C 组精华并入 `全周期实操手册`（已并入 7.1~7.4）
