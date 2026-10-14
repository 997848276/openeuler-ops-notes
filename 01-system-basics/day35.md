# day35 · systemd 资源控制与 cgroup：从分餐制到 OOM 追逐战（2026-10-14 · 128 oe-base）

## 一 侦察：版本判定 + 账本 + 枷锁三件

```bash
stat -fc %T /sys/fs/cgroup/          # tmpfs → cgroup v1（v2 是 cgroup2fs）
mount | grep cgroup | head            # freezer/net_cls/perf_event/cpuset/misc/rdma/
                                      # cpu,cpuacct/pids 各自单独挂载
cat /proc/self/cgroup                 # 13 行，v1 一进程挂多教室各记一本
sudo systemd-cgtop -b -n 2 -d 2      # 分餐大盘（第一帧冷启动不准，取第二帧）
sudo systemctl status sshd | grep -i cgroup   # Control group: /system.slice/sshd.service
```

- ★**v1 = 每个学科一间教室**（cpu/memory/pids 各管各的账本）；v2 = 一个班主任统一管五科。128 是 v1。
- **空枷锁三件套**：`cpu.cfs_quota_us=-1`、`memory.limit_in_bytes=9223372036854771712`（int64 魔法数=无限，不是病）。
- 🚩**意外发现（红线级）**：`/proc/self/cgroup` 13 行**每行都是 /system.slice/sshd.service**——SSH 会话全家（bash/sudo/cat）都住在 sshd.service 班里。**给 sshd 限资源 = 锁死自己所有 shell**。

## 二 CPU 枷锁：三层对质

```bash
# unit 里写 CPUQuota=50%
sudo systemctl show lab-hog.service -p CPUQuotaPerSecUSec   # 500ms（管理面声明）
sudo cat /sys/fs/cgroup/cpu/system.slice/lab-hog.service/cpu.cfs_quota_us   # 50000
sudo cat /sys/fs/cgroup/cpu/system.slice/lab-hog.service/cpu.cfs_period_us  # 100000
sudo systemd-cgtop -b -n 2 -d 2 | grep lab-hog              # 49.9%（运行面行为）
```

★ **`CPUQuota=50%` 落到内核 = `cfs_quota_us(50000) / cfs_period_us(100000)` = 0.5 核。**
K8s 的 `limits.cpu` 最终写的就是这批文件——同一内核机制两张脸：systemd 直接写，kubelet 替容器写。

## 三 内存枷锁与 OOM 追逐战（三回合+终局）

| 回合 | 现象 | 病因 | 教训 |
|---|---|---|---|
| ① | 99.9M 贴顶不死 | **有 swap**：`limit_in_bytes` 只限 RAM，内核先换页不杀人 | 换页活罪：工人把工具搬去隔壁仓库，活照干效率崩 |
| ② | status 显示 `swap max: 0B` 仍不死 | **管理面≠内核面**：MemorySwapMax=0 没落到 memsw 文件（swap accounting 未开） | ★"显示≠事实"新成员：systemd 说限了不代表内核真限了 |
| ③ | swapoff 后 8s 才 65.4M（不升反降） | stress-ng vm worker **memset 循环、RSS 锯齿波动**，撞线全凭运气 | 验证窗口选错=假阴性（§三十九 同族）；**非确定性工具不配下结论** |
| 终局 | python 500M 只进不出 → **91ms 被 KILL** | `signal=KILL`、journal `status=9/KILL` | OOM 猎人开枪就是 SIGKILL |

```bash
# 终局武器：确定性分配（先 swapoff -a 拆仓库）
sudo systemd-run --unit=lab-hog3 -p MemoryMax=100M -- /usr/bin/python3 -c \
  'import time; a=[]; [a.append(b"x"*1048576) for _ in range(500)]; time.sleep(600)'
```

⭐**137 之谜告破**：K8s 容器 OOMKilled 退出码 **137 = 128 + 9（SIGKILL）**——§四十一（信号）+§四十二（cgroup）+K8s §十六（OOMKilled）三线闭环。
📌 尸检分层：unit journal 只记 systemd 层；内核详细 OOM 报告（选谁/吃多少/谁开枪）在 `journalctl -k | grep -i oom`。

## 四 systemd-run 临时 unit

```bash
sudo systemd-run --unit=lab-hog3 -p MemoryMax=100M -- <命令>
# Loaded: (/run/systemd/transient/...; transient)  Transient: yes
```

- Transient unit **只存内存、不落盘、不开机自启**；死后 unit 对象自动蒸发——**journal 是唯一尸检报告**。
- 对照永久姿势：写 unit 文件 + daemon-reload；或 `systemctl set-property --runtime`（内存态）vs 不带 --runtime（落盘）——**内存态跑得快，落盘才活得久**。

## 五 坑位速查

| 坑 | 正解 |
|---|---|
| 照抄 v2 文档到 v1 | 先 `stat -fc %T /sys/fs/cgroup/` 判版本 |
| 给 sshd.service 限资源 | 🚩 SSH 会话全家都在它班里，等于锁死自己 |
| `9223372036854771712` 大数吓人 | int64 魔法数=无限 |
| status 显示 swap max: 0B 但没生效 | 管理面≠内核面；看 memsw 文件真身 |
| 限了内存进程不死越来越慢 | 有 swap=换页活罪；`free -h` 看 swap used |
| stress-ng 撞不死限墙 | vm worker 锯齿波动；换 python 只进不出 |
| cgtop 第一帧 0% | 冷启动均值，`-n 2 -d 2` 取第二帧 |
| 实验第二轮行为变了 | 前置条件（swap 等）没重置——命令组必须幂等自洽 |
| transient 死后查无此人 | 正常，自动蒸发；journal 是唯一尸检报告 |

## 面试一句话

> 「systemd 的资源控制底层就是 cgroup：CPUQuota 落到 CFS 的 quota/period 文件、MemoryMax 落到 memory 控制器——K8s 的 requests/limits 最终写的是同一批文件，同一套机制；
> 内存撞墙有关键岔路：有 swap 时内核先换页不杀人，进程活着但性能崩塌；容器之所以超限必死（退出码 137=128+SIGKILL），是因为 memsw 把 swap 一起限死了；
> 排障时管理面声明要和内核面落地对质——status 说限了不代表内核真限了；验证窗口选错、工具内存曲线锯齿波动，都会让 OOM 案变成追凶三回合——确定性结论要用确定性的工具。」
