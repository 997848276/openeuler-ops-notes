# DAY14 · A-Tune 性能调优复盘报告（2026-09-27，128 / oe-base，openEuler 24.03）

> 配套三件套手册：`openEuler命令解读与类比.md` 第 18 节（A-Tune 入门实操）。本文件是 DAY14 复盘交付物——把入门日的 A-Tune 调优尝试整理成带“前后对比框架”的报告。
> **诚实声明**：入门日因 openEuler 24.03 的 `atune-engine` 打包债（Flask 服务起而不听），profile 参数**未能真正落地**，故“调优后”数据标为**待补实测**；本报告的价值在方法论 + 基线数据 + 故障归因，而非一个漂亮的提升百分比。

## 1. 复盘目标与方法论

| 项 | 内容 |
|---|---|
| 调优对象 | openEuler 24.03 上的 nginx（Web 静态/长连接场景） |
| 工具 | A-Tune 2.x（openEuler 自带 AI 调优引擎） |
| 方法 | 装 nginx → ab 压基线 → A-Tune 识别负载 → 套用 profile → 复压对比 QPS |
| 基线 | ab：200 并发 × 2 万请求 × 5 轮 |
| 公平性铁律 | 多轮均值（单轮方差 ±10%）+ **参数层验证**（sysctl 前后快照对比），不只看 `Active` 状态位 |

## 2. A-Tune 架构（先懂构成，才懂故障）

```
atune-adm  (Go 客户端, 遥控器, 需 sudo)
   │  unix socket /var/run/atuned/atuned.sock  ← 不走 TCP！
   ▼
atuned     (Go 守护进程, 常驻主机, 6.4M, PID 36121)
   │  HTTP localhost:8383  ← 才走 TCP（engine 在这听）
   ▼
atune-engine (Python Flask + ML 大模型, 大脑, 按需起)
```

**类比**：A-Tune = 医院自动诊断系统——采集科（collector）抽血化验，AI 大夫（engine）看化验单识别**负载类型**，处方科（profile service）按病开调优处方。

## 3. 安装与自启坑（第一步就埋雷）

```bash
sudo dnf install -y atune atune-engine      # 一口气 50 个包，5 个 rpm：atune/atune-client/atune-collector/atune-db/atune-engine
sudo systemctl enable --now atuned          # 写操作必 sudo
sudo systemctl enable --now atune-engine    # ★ engine 的 preset 不自启！装完必须手动 enable
```

**踩坑**：`atune-adm` 走 **unix socket**（`/var/run/atuned/atuned.sock`，仅 root 可读写）→ 客户端命令天然要 `sudo`；而 `atune-engine` 是独立 service 且发行版 preset 不自启——只起 `atuned` 会卡在“等 engine 握手 2 分钟超时”。

## 4. 基线压测（调优前，真数据）

```bash
# nginx 起在 128；ab 压 200 并发 × 2 万请求 × 5 轮
ab -c 200 -n 20000 -t 0 http://127.0.0.1/
```

| 指标 | 调优前（基线，5 轮均值） |
|---|---|
| QPS | **≈ 3.3 万** |
| Failed requests | **0** |
| 单轮方差 | **±10%**（故对比必须多轮均值，单轮会误导） |

> 这个数字就是“前后对比”的**前**。调优的“后”本应是 A-Tune 套 profile 后复压——但见 §6。

## 5. A-Tune 调优流程（理论链路）

```bash
sudo atune-adm list          # 48 个 profile，初始全 Active=false
sudo atune-adm analysis      # 负载识别（需 engine 的 ML 大脑）
sudo atune-adm profile web-nginx-http-long-connection   # 套用 Web 长连接处方
sudo atune-adm list          # 看 Active 翻位
```

**关键认知**：`profile` 也要 engine——配方在 atuned 的 sqlite 没错，但**参数落地要打 engine 的 `/v1/profile`**（`profile.go:169` 铁证）。开处方（改 sqlite）不需要大夫，但**抓药煎药（参数落地）需要**。

## 6. 故障归因：engine “起而不听”（调优后数据缺失的根因）

| 现象 | 证据链 | 结论 |
|---|---|---|
| `atune-adm profile` 全部 refused | `atuned[36503]: Get "http://localhost:8383/v1/profile" ... refused` | engine 没在 8383 听 |
| engine 进程在（Ssl 睡眠） | `ps` 见 python3 app_engine.py 在跑 | 不是没起，是**起而不听** |
| Flask 只打 2 行无第 3 行 | 无 `Running on http://...` → `app.run()` 没跑到 | 发行版 **24.03 打包债**（time-box：入门日不还打包债） |

**诚实结论**：参数什么都没改成 → 调优效果**无法量化**，“调优后”列待补实测。这恰是运维排障的真谛：**显示≠事实**——`Active=true` 是**意向位不是结果位**（atuned 跑 profile 瞬间把意向写进 sqlite 翻 true，落地失败不回滚），验生效必须下到 **sysctl 参数层**做前后快照对比。

## 7. 坑位速查（A-Tune 六连）

| # | 坑 | 正解 |
|---|---|---|
| 1 | `atune-adm` 不带 sudo → Permission denied | 走 unix socket，天然要提权 |
| 2 | `atune-engine` 装完不自启（preset 坑） | 装完 `enable --now atune-engine`，别只起 atuned |
| 3 | `ss -tlnp` 找不到 atuned socket | 它只看 TCP；unix socket 用 `ss -xlnp`，且在子目录 `/var/run/atuned/` |
| 4 | `atuned` 等 engine 握手 2 分钟超时 | 就绪轮询：`until ss -tln | grep -q 8383; do sleep 2; done`（≤60s），不拍脑袋 `sleep` |
| 5 | `profile` 被拒 / 参数没改 | profile 也要 engine 的 `/v1/profile`，engine 不听就全 refused；参数落地失败 atuned 不回滚 |
| 6 | `Active=true` 误当“调优生效” | 它是**意向位不是结果位**；验证下到 sysctl 参数层，不看状态位 |

## 8. 前后对比框架（复压补数即可成稿）

| 指标 | 调优前（基线） | 调优后（实测/受阻） |
|---|---|---|
| QPS | ≈ 3.3 万 | **待补实测**（engine 打包债致未落地） |
| 内存占用 | — | 待补（sysctl 前后快照） |
| sysctl 关键项 | 截图存档 | 与调优前 diff |

> 复压方法：engine 修通或换可用版本 → `sudo atune-adm profile web-nginx-http-long-connection` → `sysctl -a > after.txt` 与 before 做 diff → ab 同条件复压 5 轮取均值。

## 9. 面试一句话

> “我用 A-Tune 给 openEuler 上的 nginx 做性能调优：先 ab 压出基线（200 并发 ×2 万 ×5 轮，QPS≈3.3 万、Failed=0、单轮方差±10%，所以对比必须多轮均值）。A-Tune 分三层——采集、engine 识别负载、profile 落地参数；我踩到 engine 在 24.03 上‘起而不听’的打包债，参数没真正落地。这次最大的收获不是提升数字，而是**`Active=true` 是意向位不是结果位、状态位会记意向不记结果**——验调优生效应下到 sysctl 参数层做前后快照，而不是看面板绿灯。”

## 10. 收官账

- 交付：A-Tune 调优复盘报告（架构 + 安装坑 + 基线数据 + 调优链路 + 故障归因 + 坑六连 + 前后对比框架 + 面试一句话）
- 诚实留白：engine 打包债未修，“调优后”数据待补；复压方法已在 §8 列清
- 推送：本文件传 128 改名 `day18.md`（**day14.md 已被 iSula 文章占用**，按仓库序号顺延；推前 `gops log --oneline -1` 再确认最新号），gops 三连 + 验三件套
