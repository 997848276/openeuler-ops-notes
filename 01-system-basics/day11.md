# DAY8 · A-Tune 性能调优引擎入门（2026-09-21 上午 · 128 / oe-base）

> 今天定位：**认识 + 跑通，不压测不拿数字**（数字在 DAY9）。
> 环境：openEuler 24.03（kernel 6.6.0-145…oe2403）；`atune` / `atune-engine` 两个包；atuned 常驻、atune-engine 按需。
> 意外之喜：`analysis` 报错牵出一场**四层剥洋葱排障**，比顺利跑通更值钱；结尾还识破了一个"假成功"。

---

## 一、装包起服务：依赖清单 = 架构图

```bash
sudo dnf install -y atune atune-engine
# Complete!  一口气 50 个包；五个 rpm：atune / atune-client / atune-collector / atune-db / atune-engine
sudo systemctl enable --now atuned
# Created symlink …/multi-user.target.wants/atuned.service
systemctl status atuned
# Active: active (running)  Main PID 36121  Memory: 6.4M
# Loaded: enabled; preset: disabled
```

**看依赖清单读出架构**（看它带了什么朋友，就知道它是干什么的）

| 装进来的依赖 | 暗示 |
|---|---|
| flask | engine 是个 HTTP 服务（后来实证 TCP 8383/3388） |
| numpy / pandas / scikit-learn / xgboost | "智能" = ML 推理 |
| sysstat / perf / lm_sensors | 指标采集；perf 版本号跟内核一致（6.6.0-145…oe2403） |

- **★ `Loaded: enabled; preset: disabled` 两本账**：`enabled` = 当前自启（我们 enable 的）；`preset: disabled` = 出厂默认策略。**当前状态 ≠ 出厂建议** → 这就是 dnf 装完要手动 `enable --now` 的原因。
- **★ 启动日志顺序 = 内部依赖顺序**：`valid network: ens33` → `Connecting to DB` → `initializing service: monitor` → `load profile service successful` → 判断卡在哪，**读最后一行**。

类比：A-Tune 是医院的自动诊断系统——采集科（collector）抽血化验，AI 大夫（engine）看化验单识别病种（负载识别），处方科（profile service）按病开调优处方。

---

## 二、list：48 张处方 + sudo 必需

```bash
atune-adm --help        # 不带 sudo → Permission denied
ps -ef | grep -i atune  # 只有 /usr/bin/atuned ← engine 没起的伏笔
sudo atune-adm list     # 48 个 profile，全 Active=false
```

- **为什么必须 sudo**：`atune-adm` 不走 TCP，走 **unix socket `/var/run/atuned/atuned.sock`**（仅 root 可读写）→ 客户端命令天然要提权。
- **48 个全 `Active=false`** = 出厂默认参数，正好当 **DAY9 压测基线**。
- **命名三段式**：`<业务域>-<应用>-<压测模型>-<介质>`，如 `web-nginx-http-long-connection`、`database-mysql-2p-sysbench-ssd`；业务域约 12 类（web/database/big-data/storage/virtualization/redis/middleware/encryption/hpc/cloud-compute/docker/basic-test-suite）。

---

## 三、三层架构 + 通信双通道（排障前先挂图）

```
atune-adm (Go 客户端, 遥控器)
   │  unix socket /var/run/atuned/atuned.sock  ← 不走 TCP！
   ▼
atuned (Go 守护进程, 常驻主机, PID 36121, 6.4M)
   │  TCP 8383(collector) / 3388(engine) + gRPC/HTTP(S)，证书 rest_certs / engine_certs
   ▼
atune-engine (Python Flask + ML, 大脑, 按需)
```

---

## 四、analysis 报错：四层剥洋葱（07:48~08:14）

**第一层 · TLS 证书缺失**

```bash
sudo atune-adm analysis
# open /etc/atuned/rest_certs/ca.crt: no such file or directory ; rpc error …
ls /etc/atuned/rest_certs/ /etc/atuned/engine_certs/   # 目录在，全空
rpm -ql atune | grep cert                              # rpm 只打包目录，不发证书
```

- **这是安全设计不是 bug**：私钥/信任材料不该由 rpm 预置，生成权留给部署者（"对的固执"）。
- **修法**：备份两 cnf → `sed` 把 `rest_tls` / `engine_tls` 改 `false`（实验室 OK；生产要正经签发证书）。

**第二层 · atune-engine 服务压根没起**

```bash
sudo systemctl enable --now atune-engine   # python3 进程 36492
sudo systemctl restart atuned              # 36503
# 报错 URL 从 https://…:8383 变 http://…:8383 ← TLS 层翻过去的旁证
```

- ⚠️ **验证 grep 忘带 sudo → Permission denied = 验证白做**。新警戒线：**验证命令自己报错 ≠ 验证通过**。

**第三层 · connection refused：8383 无人听**

```bash
sudo atune-adm analysis
# Post "http://localhost:8383/v1/collector": dial tcp [::1]:8383: connect: connection refused
sudo ss -tlnp | grep -E ':8383|:3388'   # 空！两个 TCP 端口都没人听
curl -v http://127.0.0.1:8383           # → 000（没拿到任何响应 = refused 的孪生）
sudo ss -xlnp | grep atuned             # /var/run/atuned/atuned.sock ← unix socket 在这
journalctl -u atuned | tail             # 每次启动 +2min：waiting for pyservice timeout
```

- **★ `ss -tlnp` 只看 TCP，unix socket 要 `ss -xlnp`**；glob `ls /var/run/*.sock` 不递归，socket 在子目录里。
- **collector 采集本身成功**（CPU/MEM/NET ens33…全采到）→ **故障不在"看"，在"传"**。
- atuned 等 engine 握手 2 分钟超时 → engine 起了但没"报到"。

**第四层 · engine 起而未听：5432 无罪释放（08:14）**

```bash
sudo systemctl cat atune-engine        # 怪象：连 sudo 都 Permission denied → 换 sudo cat 直读
ps -o pid,stat,wchan:32,etime,args -p 36492
# /usr/bin/python3 /usr/libexec/atuned/analysis/app_engine.py /etc/atuned/engine.cnf  STAT=Ssl
find /var/log -iname '*atune*' -o -iname '*engine*'   # 无 → journal 3 行 = 全部口供
sudo cat -n /etc/atuned/engine.cnf
# [database] db_enable = false   # default is false  ← 5432/PostgreSQL 无罪！
```

- C-4 看到 `db_port=5432` 一度怀疑要装 PostgreSQL —— **全读配置定案 `db_enable=false`，端口根本不用**。
- ⭐ **嫌疑犯靠证据定罪，也靠证据释放；grep 片段 ≠ 全读配置**。当时拍脑袋装一套 PG 就白装了。
- Flask 日志只打两行（`Serving Flask app` / `Debug mode: off`），**没有第三行 `Running on http://…`** → `app.run()` 没跑到，HTTP 服务根本没起。进程 Ssl 睡眠 + 配置无罪 → **24.03 这版 atune-engine 自己的打包债**（time-box：入门日不还打包债）。
- 顺手收获：engine.cnf 的 **`[bottleneck]` 阈值表**（CPU util 80 / mem 70 / net 70 / disk 70 / perf_ipc 1）= A-Tune 判瓶颈的标尺。

---

## 五、profile 验尸：判决反转——它也要 engine（08:19~08:41）

推理（后被推翻）：analysis 是"诊断"（要 engine 的 ML 识别）；profile 是"开处方"（配方存 atuned 的 sqlite，启动日志 `load profile service successful` = 开柜门）→ 以为**开处方不需要 engine**。

```bash
sudo atune-adm profile web-nginx-http-long-connection
# Bios        ← 绿
# Bootloader  ← 红     ← 客户端零报错、秒回 → 当场误判"绕行成功"
sudo atune-adm list    # 48 个仍全 false ← 第一丝不对劲
```

**journal 验尸翻案（08:20:58，正是跑 profile 那一刻）**

```
atuned[36503]: level=error msg="Get \"http://localhost:8383/v1/profile\":
  dial tcp [::1]:8383: connect: connection refused" file="profile.go:169"
（同一秒重试几十次）
atuned[36503]: level=error msg="model monitor module cpu get cpu_info data failed:
  Get \"http://localhost:8383/v1/monitor\": … connection refused" file="monitor.go:81"
```

**判决**
- **profile 也要 engine**：配方在 atuned 的 sqlite 没错，但**参数落地要打 engine 的 `/v1/profile`**（`profile.go:169` 铁证）→ engine 不听 → 全部 refused → **什么都没改成**，`Active` 保持 false。（类比修正：开处方不需要大夫，**抓药煎药需要**。）
- 客户端那两行 `Bios`/`Bootloader` 只是**尝试过的调优层条目名**（固件层/引导层），不是成功清单——atuned 把失败吞进后台重试，客户端根本不知道。
- 彩蛋：`/v1/monitor` 也 refused —— monitor 模块持续要 engine 喂数据 → **engine 一挂，诊断+调优+监控整条智能链全瘫**。
- **意外的好消息**：什么都没改成 → 48 个 profile 干净 `false` = **DAY9 压测基线天然纯净，无需还原**。

**★ 教训（与 9/16 SSH 那条配成一对）**

| 日期 | 教训 |
|---|---|
| 9/16 | **客户端报错 ≠ 服务端真相**（客户端说 refused，服务端可能在别处） |
| 9/21 | **客户端不报错 ≠ 服务端成功**（异步/火后不管型任务，错误全吞在服务端日志） |

★ **验"操作是否生效"永远看状态位（`Active=true`）或服务端日志，不能只看"命令没报错"**。

---

## 六、今日坑清单（汇总）

| # | 坑 | 一句话 |
|---|---|---|
| 1 | rpm 只打包空证书目录 | 证书自己发；实验可 sed 关 TLS，生产要正经签发 |
| 2 | atune-engine 是独立 service 且 preset 不自启 | 装完要 `enable --now atune-engine`，别只起 atuned |
| 3 | 验证命令忘 sudo | **验证命令自己报错 ≠ 验证通过** |
| 4 | `ss -tlnp` 找不到 socket | 它只看 TCP；unix socket 用 `ss -xlnp`，且在子目录 `/var/run/atuned/` 里 |
| 5 | 见 `db_port=5432` 就要装 PG | **grep 片段 ≠ 全读配置**，先看 `db_enable` |
| 6 | Flask 打印两行就以为起好了 | 有没有第三行 `Running on` 才是"真的在听" |
| 7 | 多行 `\` 续行粘贴断裂 | 反斜杠后带空格即失效 → 单行更稳 |
| 8 | 客户端秒回无报错就当成功 | **客户端静默 ≠ 成功**：profile 零报错秒回，journal 里 refused 刷屏、`Active` 纹丝不动——验 `Active`/journal 才算数 |

---

## 七、面试话术

**① 架构与通信**
> "A-Tune 是 openEuler 的自动调优框架，三层架构：atune-adm 客户端通过 unix socket 把命令发给常驻的 atuned，atuned 再通过 TCP/gRPC 调 Python 写的 atune-engine 做 ML 负载识别。它的依赖也很直白——flask 负责通信、numpy/pandas/sklearn/xgboost 负责 ML 推理、sysstat/perf/lm_sensors 负责采集。"

**② 部署排障（最值钱）**
> "我实际部署踩了四层坑：rpm 不发 TLS 证书要自己关或签、engine 是独立服务要单独 enable、8383 连接拒绝要用 ss 分清 unix socket 与 TCP、最后 engine '起而未听'时靠全读配置排除了 5432 数据库嫌疑——**排障要剥洋葱，每层用证据说话，不靠猜**。最有意思的是 profile 命令：客户端零报错秒回、看似成功，journal 里却刷屏 connection refused、`Active` 纹丝不动——配方虽存 atuned 本地 sqlite，参数落地仍要打 engine 的 `/v1/profile`。所以我总结了一条：**验调优是否生效，永远看 `Active` 状态或服务端日志，不能信客户端没报错**。"

