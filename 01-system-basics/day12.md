# DAY9 · A-Tune 实战压测与"不造假的数字"（2026-09-22）

> 机器：128（oe-base，openEuler 24.03 LTS，kernel 6.6.0-145，SELinux Enforcing）
> 主题：nginx + ab 压测基线；A-Tune engine 排障续集；压测方法论（对照组 / 参数层验证 / 多轮均值）

## 一、任务与结局速览

- 计划要求：装 nginx → ab 压基线 → A-Tune 调优 → 对比 QPS → 简历写"QPS 从 X 提升到 Y"。
- 实际结局：**基线拿到；调优没成（engine 打包债）；识破一次"假提升"**——换来比假 Y 硬十倍的方法论。
- 诚实数字：5 轮有效压测 QPS 30042~35940，**均值 ≈3.3 万，Failed 全 0，单轮方差 ±10%**。

## 二、压测三件套

```bash
sudo dnf install -y nginx httpd-tools   # nginx=web；httpd-tools 里是 ab
sudo systemctl enable --now nginx
ab -n 20000 -c 200 http://127.0.0.1/    # 200 并发 × 2 万请求，~0.7s 跑完
```

- 意外发现：nginx 已装好且 active——DAY6 firewalld services 里 `http` 的伏笔。**侦察先行，避免重复劳动**。
- 类比：ab 是"雇 200 个客户同时进门下单 2 万次"，QPS = 柜台每秒接单数。

**基线回显（X = 30042.03）**

```
Requests per second:    30042.03 [#/sec] (mean)
Time per request:       6.657 [ms] (mean)
Time per request:       0.033 [ms] (mean, across all concurrent requests)
Failed requests:        0
95% 11ms / 99% 16ms / 100% 28ms
```

**★ 两行 Time per request 的区别（ab 最常被问）**：
- `6.657 ms` = 并发÷QPS（200/30042）——**客户视角**：一个请求从发出到完成等了多久（含排队）
- `0.033 ms` = 1÷QPS——**柜台视角**：服务器平均每 33µs 吐出一个请求
- 两个都对，视角不同。

## 三、engine 排障续集：三方对照（前台 / systemd / nohup）

| 尝试 | 现象 | 结论 |
|---|---|---|
| systemd 起（DAY8 遗留） | active 但 8383 不听，analysis refused | 起而不听 |
| sudo 前台跑 app_engine.py | Flask `Serving` 两行后挂住，**无 traceback** | 代码没坏 |
| `nohup` 后台 + sleep 3 验证 | ss 空、analysis refused → 以为失败 | **其实只是没等够** |
| 10 分钟后再前台跑 | **`Address already in use: Port 8383`** | nohup 引擎当时已占 8383（ML 模型加载要几十秒） |
| 再往后 `ss` 看 8383 | **又空了**（ps 有进程 18793、ss 无监听） | **活着 ≠ 听着**，打包债实锤 |

**两个方法论沉淀**：

1. **就绪轮询，不拍脑袋 sleep**——9/20 mysql 备份竞态（Persistent 补跑早 mysqld ready 7 秒）、DAY8 atuned 握手 timeout、今天 sleep 3，同一个病第三发：

```bash
until ss -tln | grep -q 8383; do sleep 2; done   # 循环等就绪，封顶 60s
```

2. **前台 / systemd 行为不一致 → 先查环境差异**：SELinux 域（`ps -eZ`）、unit 沙箱（`systemctl cat`）、环境变量。本次引擎行为诡异（能绑后来又不绑），定性发行版打包债，留周末尸检：`tail /tmp/engine.log`（nohup 重定向的遗言）+ `ausearch -m avc`。

## 四、★★ 假 Y 事件（本日核心）

时间线：第二轮 ab 跑出 **35940（比基线 +19.6%）**，险些当成 A-Tune 调优战果写进简历。

**三连判决**：

1. **35940 是假 Y**：`profile` 只回显 `Bios`/`Bootloader` 红字（DAY8 同款假成功签名）；sysctl 六项前后快照**全等**——变量根本没动，+19.6% 纯属**运行方差**。

```
kernel.sched_autogroup_enabled = 0    ← 前后全等
vm.swappiness = 30                    ← 前后全等
net.ipv4.tcp_tw_reuse = 2             ← 前后全等
net.core.somaxconn = 4096             ← kernel 5.4+ 默认值，不是调优证据
net.ipv4.tcp_max_syn_backlog = 256    ← 前后全等
fs.file-max = 9223372036854775807     ← 前后全等
```

2. **`Active=true` 是意向位不是结果位**：DAY8 判决"Active 保持 false"错了一半——复查发现 `web-nginx-http-long-connection` 早翻 true 了。atuned 跑 profile 那一刻就把意向写进 sqlite，参数落地失败**不回滚**。验生效必须下到参数层。
3. **没有对照组的数字会撒谎**：对比成立的前提 = 变量真的变了（参数层证明）+ 多轮取均值。

**全天 5 轮**：30042 / 35940 / 31957 / 34318 / 32652，Failed 全 0。

## 五、坑（必记）

| # | 坑 | 一句话 |
|---|---|---|
| 1 | `sleep 3` 就去验服务 | 初始化要几十秒 → 就绪轮询，不拍脑袋 sleep |
| 2 | `Address already in use` 当灾难 | 它是线索：端口刚被人占过，顺藤摸瓜 |
| 3 | `somaxconn=4096` 当调优证据 | kernel 5.4+ 默认就是 4096，默认值 ≠ 调优值 |
| 4 | sudo 密码敲错后继续跑 | "Sorry, try again" 后那条命令没执行，验证白做——回显逐条对 |
| 5 | profile 回显 Bios/Bootloader 红字 | 假成功签名（尝试过的条目名，非成功清单） |
| 6 | 单轮 QPS 当真 | 单轮方差 ±10%，对比必须多轮均值 |
| 7 | 前台 / systemd 行为不一致 | 先查 SELinux 域 / unit 沙箱 / env |
| 8 | 进程活着 = 服务在听 | ps 有进程、ss 无监听可以同时成立 |

## 六、面试一句话

> "我用 ab 对 openEuler 上的 nginx 做过基线压测：200 并发 2 万请求，QPS 均值约 3.3 万、Failed 0。最值钱的不是数字，是过程中识破了一次'假提升'——第二轮快了 19.6%，但 sysctl 前后快照证明一个参数都没变，纯属方差。所以我的压测纪律是：对照组 + 参数层验证 + 多轮取均值。另外排查 A-Tune engine 时发现它前台能起、systemd 下'起而不听'，三方对照定位成发行版打包债——数字可以不好看，但每个数字都要能自证清白。"

## 七、留存物与环境现状

- `/tmp/sysctl-before.txt`（调参前快照）、`/tmp/engine.log`（nohup 引擎输出，尸检素材）
- nohup 引擎（PID 18793）还挂着；systemd 的 atune-engine 处于 stopped
- 收尾建议：`tail -30 /tmp/engine.log` 看遗言 → `sudo kill 18793 18792 18790` → `sudo systemctl start atune-engine`（恢复 DAY8 现场基调）→ `sudo chronyc makestep`（VM 时钟慢 ~7h，顺手校）
