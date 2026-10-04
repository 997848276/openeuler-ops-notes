# day25 · 系统调优与资源限制（128 oe-base · openEuler 24.03 · cgroup v1）

> 主题：ulimit/limits.conf、sysctl、systemd 资源限额。核心认知：限额有两套供给链、内核参数有两条生效轴、cgroup 有两个版本——配了不生效，先问"谁发额度"，再问"临时还是永久"。

## 一、侦察基线

- 登录 shell `ulimit -n` = 1024（soft）；limits.conf/limits.d 全空白
- sysctl 现代化：`vm.swappiness=30`、`fs.file-max=9223372036854775807`（int64 顶格，老口诀作废）、`fs.nr_open=1073741816`、`somaxconn=4096`、port_range 32768-60999
- `stat -fc %T /sys/fs/cgroup` = tmpfs → **cgroup v1**（systemd 255 ≠ 必配 v2）
- ★对比炸弹：`systemctl show sshd -p LimitNOFILE` = 524288 vs 登录 shell 1024 → 512 倍鸿沟、两套供给链

## 二、PAM 侧：limits.conf 只管登录会话

- `/etc/security/limits.d/99-nofile.conf` 写 `* soft/hard nofile 65535` → 当前 shell 纹丝不动（1024/524288）
- 生效前提：PAM 挂 `pam_limits.so`（system-auth 直挂；su 经 include 间接挂）
- `su - devops` 新会话 → soft=65535 hard=65535 ✅
- ★坑：通配符 `*` 不作用于 root（sudo 也过 PAM，root 照样旧值）；验证一律用普通用户
- 类比：limits.conf = HR 入职手册，只在新员工报到时生效

## 三、systemd 侧：服务不吃 limits.conf

- limtest.service 实测：journal 输出 `NOFILE soft=1024 hard=524288`（一眼不看 limits.conf）
- 药方 = drop-in：`/etc/systemd/system/<unit>.service.d/*.conf` 写 `LimitNOFILE=65535` + daemon-reload → 65535 ✅
- `systemctl cat <unit>` = 主文件 + drop-in 拼装实物
- 全局开关：`/etc/systemd/system.conf` 的 `DefaultLimitNOFILE=`

## 四、sysctl 双生效轴 + 句柄三本账

- `sysctl -w` 临时（/proc/sys 真相源）vs `/etc/sysctl.d/*.conf` 永久 + `sysctl --system` 重放
- ★删配置行 ≠ 内核值回默认（写入即驻留）；归位必须显式 `-w` 写回
- 层级：单进程 ulimit ≤ fs.nr_open ≤ fs.file-max；`file-nr` 三列 = 已分配/空闲/上限
- too many open files 九成是单进程 ulimit 的锅

## 五、systemd 资源限额

- `systemctl set-property <unit> CPUQuota=50% MemoryMax=100M TasksMax=50`
- 自动生成 drop-in 到 `/etc/systemd/system.control/`（托管区 Do not edit）；`--runtime` 则写 /run 重启失效
- ★`systemctl revert` = 撤 /etc 下该 unit 全部 drop-in + 本地覆盖回 vendor 默认（连手写的一起撤）
- CPUQuota 分母是单核

## 六、内存双卡连环案（cgroup v1 四连翻车结案）

1. MemoryMax=100M + 300M → 活（v1 只卡 RAM，swap 无限兜底）
2. + MemorySwapMax=0 → 还活（systemd 在 v1 静默忽略，memsw 恒 infinity）
3. 三层对质（show 声明 / cgroup 文件落地 / usage 行为）→ RAM 卡真在工作（usage 顶格 100M）
4. 手写 memsw 卡 EBUSY → ★memsw=RAM+swap 总账，限值不得低于当前用量
5. 结案：ExecStartPre 在 usage≈0 时抢写 memsw → `Memory cgroup out of memory: Killed process` 五连 ✅

启示：v1 内存语义坑 → 容器圈推 v2 的实权理由；v1 想真卡走 Docker `--memory-swap`（亲手写 memsw 文件）

## 七、坑位速查

| 现象 | 正解 |
|---|---|
| limits.conf 改了没生效 | 只对新 PAM 会话生效；服务走 unit LimitXXX |
| `*` 条目对 root 无效 | 通配符不作用于 root，用普通用户验证 |
| 删 sysctl 配置行参数没回默认 | 写入即驻留，显式 `-w` 归位 |
| CPUQuota=50% 以为限整机 | 分母是单核 |
| revert 以为只撤 set-property | 撤全部 drop-in 回 vendor 默认 |
| MemoryMax 限了进程不死 | v1 只卡 RAM；手写 memsw 卡要抢在用量前（EBUSY=限值低于总账） |

## 面试一句话

> "调优先分清谁发额度、两条生效轴：登录会话走 PAM/limits.conf，systemd 服务走 LimitXXX/drop-in；层级 ulimit ≤ nr_open ≤ file-max，现代内核 file-max 顶格，瓶颈都在单进程级。我实战踩过 cgroup v1 的 MemoryMax 只卡 RAM、MemorySwapMax 被静默忽略的坑，靠『管理面声明 → 内核面落地 → 运行面行为』三层对质加 ExecStartPre 抢写 memsw 卡结案——容器圈推 v2 不是赶时髦，是 v1 内存语义真的坑。"
