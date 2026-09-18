# DAY5 · systemd 编排（2026-09-18）

> 机器：128（oe-base）｜systemd 255｜10:39 开机 0 分钟即开工。
> 主线：**手写 service → 主动复现 203/EXEC → 谋杀与自愈 → 开机体检 → 手写 timer**。
> 定位：254 上已经用过 timer（备份），今天从"用过"升级到"能手写 + 懂原理"。

---

## 一、块 0 · 体检 + 一个伏笔

- `systemctl get-default` = **multi-user.target**（服务器无图形）
- enabled 名单里**没有 nginx**——昨晚它明明活着：手动 start 没 enable，**开机不自启**，这次已暴毙。反面教材认证
- **`/data` 回到 noquota**：昨晚配额实验不留痕的承诺，重启自动兑现 ✅

## 二、块 1 · 手写第一个 service（四金蛋）

**纪律：脚本先裸跑再交给 systemd**（`sudo timeout 12 /usr/local/bin/myapp.sh` 吐 3 行日志 = 脚本没病）。

```ini
[Unit]
Description=My First Hand-written Service (DAY5)
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/myapp.sh
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

daemon-reload → `enable --now` → **三绿**：`Loaded: enabled` + `Active: active (running)` + `Main PID`。

**四个金蛋（全从回显里挖的）**
1. **enable = 建符号链接**：`Created symlink /etc/systemd/system/multi-user.target.wants/myapp.service → ...` —— WantedBy 的全部秘密：往 target 的 wants/ 目录塞 symlink，"开机必点名名单"写名字
2. **preset vs enabled**：出厂建议 vs 实际状态，互不矛盾
3. **CGroup 进程树**：`/system.slice/myapp.service` 下 bash + sleep 全挂——systemd 靠 cgroup 认领全家，`systemctl kill` 一锅端的底气
4. **journal 收编 stdout**：脚本只管 echo，journald 全记录（`myapp.sh[1954]:` 前缀）；要落盘轮转才自管日志（254 备份脚本的 `exec >>` 模式）

## 三、主动复现 203/EXEC + mv 标签坑（本日最值钱）

**复现**：mv 脚本到 /root → 改 ExecStart → start：

```
Active: activating (auto-restart) (Result: exit-code)     ← 不是 failed！Restart=always 的签名
Process: 2059 ExecStart=/root/myapp.sh (code=exited, status=203/EXEC)
journal: Scheduled restart job, restart counter is at 4.   ← 死循环重试中
```

**增强坑：恢复路径后依然 203/EXEC！**
sed 改回 `/usr/local/bin` 还是炸——**`mv` 是 rename，SELinux 标签跟着文件走**：admin_home_t 从 /root 原样带回 bin_t 的地界。目录对没用，systemd 查文件自己的标签。

**修复（亲眼看改标签）**：

```
$ sudo restorecon -v /usr/local/bin/myapp.sh
Relabeled ... from ...admin_home_t:s0 to ...bin_t:s0
$ ls -Z ...        # unconfined_u:object_r:bin_t:s0
→ 三绿复活
```

**SELinux 三部曲**：①放对目录（254 第一课）→ ②mv 会带标签跑（128 今天）→ ③`restorecon` 唯一终点。

## 四、谋杀与自愈

```
systemctl show -p MainPID --value myapp     # 2259（程序化取值，脚本里好用）
sudo kill -9 $(systemctl show -p MainPID --value myapp)
sleep 6 → Main PID: 2299，Active since = 刚刚    ← 换人上岗
```

- `systemctl stop`：它安安静静不重试 —— **管理员意志，systemd 不违抗**
- `kill -9`：journal `code=killed, status=9/KILL` → `Scheduled restart job` 秒救
- **Restart 策略只对"非正常死亡"负责**

## 五、开机体检：systemd-analyze

```
Startup finished in 1.763s (kernel) + 2.936s (initrd) + 5.774s (userspace) = 10.474s
multi-user.target reached after 4.340s in userspace.
```

- **blame 口径课**：榜首竟是 `myapp.service 3.016s`——blame 统计"boot 后所有启动活动"（含手动 start 和 203 死循环），数字异常先查该 unit 最近经历过什么。ttyS0~S3 串口 2.7s = VMware 陪跑员
- **critical-chain**：`@`=到达时刻，`+`=自身耗时；链上最大堵点 `tuned +1.400s`
- **`data.mount @1.632s +216ms` 在关键链上** —— 早上 fstab 那行（DAY4）在开机链条会师
- **优化开机只看：`+` 大、且在链上**。blame 榜高但不在链上的，改了白改

## 六、手写 timer（收官战）

oneshot service + timer：

```ini
[Timer]
OnBootSec=1min          # 单调式①：开机后 1 分钟首跑
OnUnitActiveSec=1min    # 单调式②：之后每活跃 1 分钟
[Install]
WantedBy=timers.target   # timer 归 timer 家族
```

**实测时间线**：
- `10:50:39` enable **瞬间即跑**——OnBootSec 的触发点（开机+1min）早过了，**错过的单调触发点，启用时立刻追账**
- `10:52:21` 第二次——**距上次 102 秒不是 60**：**AccuracySec 默认 1min**，触发在窗口内浮动（合并唤醒省电）；要秒级精度写 `AccuracySec=1s`
- 两种时间语法：**单调式**（OnBootSec/OnUnitActiveSec）vs **日历式**（254 的 `OnCalendar=*-*-* 02:30:00`）
- 清理对称：`Removed .../timers.target.wants/df-snap.timer`
- 系统自带 timer：`dnf-makecache.timer`（12min）、`systemd-tmpfiles-clean.timer`（每天）

## 七、踩坑清单（8 条）

1. 改 unit 不 `daemon-reload` = 白改（systemd 有缓存）
2. 脚本没裸跑就交给 systemd → `timeout 12` 先验尸
3. **mv 搬文件带 SELinux 标签** → `restorecon` 才是终点
4. Restart=always 时状态是 `activating (auto-restart)` 不是 failed → 判决书看 `Process:` 行
5. `systemctl stop` 不触发 Restart（只管非正常死亡）
6. blame 榜数字异常 → 先查该 unit 最近经历过什么
7. timer 默认 AccuracySec=1min → 要准写 `AccuracySec=1s`
8. timer 清理三件套：disable --now + 删文件 + daemon-reload

## 八、面试话术（3 条）

1. **"手写过 service 吗？"** → 三段骨架脱口而出：[Unit] 依赖（After=）、[Service] 启动自愈（Type/ExecStart/Restart）、[Install] 挂靠（WantedBy）。enable 的本质是 target.wants/ 下建 symlink；改完必 daemon-reload。
2. **"服务起不来怎么排？"** → status 看 Loaded/Active/MainPID，journalctl -u 看日志；`status=203/EXEC` 先查 SELinux（脚本目录 + restorecon）；`activating (auto-restart)` 说明 Restart=always 在重试，判决书在 Process 行。
3. **"定时任务用什么？"** → systemd timer 替代 cron：和 service 绑定成对、journald 收日志、Persistent 补跑；OnCalendar 日历式、OnBootSec/OnUnitActiveSec 单调式；注意默认 AccuracySec=1min 的触发浮动。
