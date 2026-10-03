# DAY24 · 性能排查四件套：CPU / 内存 / 磁盘 IO / 网络（2026-10-03 · 128 oe-base + 254 oe-docker）

> 主题：用 stress-ng / fio / iperf3 **造故障 → 定位 → 归因 → 收敛**，四象限全闭环。
> 环境：128（oe-base，2 核 3.3G，openEuler 24.03，SELinux Enforcing）+ 254（oe-docker，Docker 机）。

## 0 · 基线哲学：先记下"没病的样子"

- 工具清点：sysstat（iostat/sar/pidstat）已装，补装 stress-ng 0.15.03 / fio 3.34 / iperf3 3.18。
- 基线：load `0.00 0.00 0.00`｜Mem used 892Mi / available 2.4Gi / Swap 0B｜nproc=2｜/ = ext4（≠ /data 的 xfs）｜**swappiness=30**（openEuler 实测，非 60）。
- 坑：`dnf install` 的 "Nothing to do" = 幂等确认，与"验证靶子 Nothing to do 不算"两个语境。

## 1 · CPU（stress-ng --cpu）

- 烧满 `--cpu 2`：vmstat `r=2 / us=100 / id≈0`；top `%Cpu 100.0 us`，stress-ng-worker 霸榜；**pidstat CPU 列 0/1 = 分居两核**；load 爬 0.29→1.83。
- 超载 `--cpu 4`：`r=6`（新旧组并存）、load 冲 2.93 —— **load > nproc = 排队实锤**。
- 熄火：load `0.00 0.05 0.20` —— **1/5/15 分钟 = 指数平均，退烧有过程**。
- ★坑：后台 sudo 触发 SIGTTIN → **T 态 Stopped (tty input)**，密码被前台 bash 吃掉 → 正解 = 前台 `sudo -v` + 后台 `sudo -n`。
- ★坑：**vmstat 首行 = 开机以来均值**，永远跳过首行。

## 2 · 内存（stress-ng --vm）

- **物理没吃满也 swap**：used 1.66G/3.3G 换出 376Mi —— swappiness 主动腾匿名页保 page cache（策略非故障）。
- 重压叠加：swap 435Mi、**si/so 双高 = 页乒乓**、**sy 飙 30~78%（换页管理是内核的活）**。
- **VIRT ≠ RES**（550M vs 200~500M）；看内存认 RES/%MEM，手头现金认 available。
- 收尾：内存回落、**Swap 435→94Mi 慢慢回落 + si=1** —— Swap 是被动回收（借了慢慢还）。

## 3 · 磁盘 IO（fio --direct --time_based）

- dd 4G direct 2 秒完（VM 后端 SSD）→ **fio `--time_based --runtime=60` 定长压制**。
- `lvs -o +devices`：data_lv = **sdb(0) + sdc(0) 分段 linear**。
- 实测：dm-3 `%util 92~94%、1.26GB/s、w/s 1233`；**sdb 94.5% / sdc 全程 0%** —— **跨盘拼池 ≠ 条带加速**（分流要 `lvcreate -i 2` 或 RAID0）。
- 请求拆分：LV 层 1M（wareq 1024）→ 物理盘 512K×2。
- **CPU 签名 = sy 49.7% + hi 5.1%，iowait 仅 4.6~13.7%** —— IO 瓶颈认 await/%util/aqu-sz/b 列，wa 低 ≠ IO 闲。
- 坑：`--direct=1` 必加（否则进 page cache，iostat 看不见）；rm root 文件 denied = **父目录 w 权限位**问题。

## 4 · 网络（iperf3 跨机）

- 速率三部曲：回环 **90.8G** → 自连 **92.4G** → **真跨机 4.48G**（Retr 236 集中在慢启动，之后归 0）。
- **量级 = 身份验钞机**：G 级 = 没跨机；`local <IP>` 与 server 不同机才是跨机。
- 4.48G ≠ 1G：**VMware 同宿主机虚拟交换 = 内存级转发，上限由网卡型号决定**（vmxnet3 不限 1G）。
- `No route to host` = **host-prohibited**（服务端 firewalld REJECT 陌生端口）——DAY6 排障三连现场命中。
- firewalld runtime 放行：立即生效、reload 消失，测试端口首选。
- ★防串机口诀：**敲命令前先看提示符 @ 后面的主机名**（今天身份三连串全靠它破案）。

## 坑位速查表

| 现象 | 正解 |
|---|---|
| `sudo … &` 卡 Stopped | 后台无 tty（SIGTTIN）；前台 sudo -v + 后台 sudo -n |
| load 熄火后不归零 | 指数平均，退烧有过程 |
| vmstat 首行假空载 | 首行 = 开机均值，跳过 |
| 没吃满也 swap | swappiness 主动腾页（=30），是策略 |
| dd 秒完抓不到 | fio --time_based 定长压制 |
| 打盘无流量 | 忘 --direct=1 |
| 拼池只烧一块盘 | linear 非 striped；-i 2 或 RAID0 |
| wa 低但 IO 忙 | 快盘签名 = sy+hi；认 await/%util |
| iperf3 No route to host | 服务端 firewalld REJECT；runtime add-port |
| iperf3 跑出 90G | 回环/自连；local 与 server 同机 |

## 面试一句话

> "服务器卡了我按四象限分诊：先 uptime 拿 load 对比核数，top 看 CPU 烧在谁头上——us 高是用户进程、sy/hi 高是内核和中断、wa 高才指向磁盘；free 看 available 和 si/so，物理没满也会 swap（swappiness 主动腾页）；iostat -x 看 %util/await 定位到具体盘，我实测过 LVM 跨盘拼池是 linear，顺序写全砸 sdb、sdc 旁观——拼池不等于分流；网络用 iperf3 压，回环 90G、跨机 4.48G，量级一眼识破身份。核心纪律两条：先记基线再下结论，load 高不一定是 CPU——D 态也算 load。"
