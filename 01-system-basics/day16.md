# DAY16 · 第 2 周复盘（9/21–9/27）

> 第 2 周主题：openEuler 原生工具链 + 监控加餐线。本篇收拢五样：战果总览、坑位库精选、跨线方法论、面试金句汇总、下一步。

## 一、战果总览

| 主题 | 产出 | 代表坑 |
|---|---|---|
| iSula 容器引擎（iSulad 2.1.5） | 128 采集端接入监控 + web 容器 host 网络真收官 | `-p` 静默不执行；mirrors bug 绕道直连 |
| 监控全链路（计划外加餐） | Node Exporter×2 → Prometheus（3 target up）→ Grafana（自建三面板 + 1860）→ Alertmanager 告警出师 | evaluation_interval 默认 1m |
| EulerCopilot（理解向） | 定位/架构/能力串线笔记（day15） | 不是单机 rpm，是 K3s + 大模型 API 服务栈 |

## 二、iSula 关键认知（day14）

- **`-p` 静默接受不执行**：普通 isula run 容器只有 lo 一块网卡；CNI 三件套只服务 CRI Pod/sandbox；`isula ps` 里 `0.0.0.0:8080->80` 是配置显示、非事实
- **正解 `--net=host`**（生产标准姿势之一）
- **200 OK 假阳性案**：宿主 nginx 抢答 80 端口 → 鉴别三判据：Server 版本号 + `sudo ss -tlnp` 看进程名 + 容器日志 → **验证要验到"应答者身份"才算数**

## 三、监控线关键认知（9/24 加餐）

- Node Exporter 三件套：`--pid=host` + `-v /:/host:ro` + `--path.rootfs=/host`（透视宿主，否则数据假）
- 加新节点三步：新机装 exporter + 开 9100 → yml 加 target → restart（生产升级向：file_sd + rich rule 限源）
- **PromQL 的家在浏览器 Graph 查询框**，贴进 bash = `syntax error near '('`
- Grafana：**一面板一查询** / 改查询先 Run queries（y 轴量级是残影照妖镜）/ Grafana 11 无 Apply 按钮
- 1860 导入：Pressure(PSI) No data = 正常；Last 24h 左侧空白 = 监控无回看
- **告警机制**：`evaluation_interval` 默认 1m（非 scrape 15s）→ pending 不发 Alertmanager，**firing 才敲门**；取证时点要过「评估周期 + for 时长」双重门
- alertmanager 地址写**宿主 IP** 不写 localhost（容器网络第三课）
- 监控数据彩蛋：Uptime 面板反推宿主最近开机时间，可做交叉验证

## 四、跨线方法论（比命令更值钱）

1. **显示 ≠ 事实**：HTTP 200 ≠ 容器在应答，验到应答者身份才算数
2. **阴性证据也是证据**：警告消失 / 文件消失 / 日志无新增行，都是线索
3. **客户端报错 ≠ 服务端真相**；**客户端静默 ≠ 成功**（验生效看状态位/journal，不看命令脸色）
4. **同形不同因**：`refused` 可能是没人听也可能是 REJECT，须 `ss` 排除
5. **git 铁律**：全程 `sudo git -C /root/openeuler-ops-notes`——3 次在 ~ 裸跑踩坑后，立 `gops` 别名根治：
   `alias gops='sudo git -C /root/openeuler-ops-notes'`

## 五、面试金句汇总（第 2 周全量）

**看板版**

> "监控我不止会搭，还会变现成看板：Prometheus 采集两台 openEuler，Grafana 导入数据源自建三面板（CPU/内存/负载按 host 标签双机对比），能从 load1 曲线读出镜像拉取时刻的单核满载事件——我清楚 load1=1 的含义、聚合标签的取舍，也踩过 Grafana 11 取消 Apply 按钮和查询残影的坑，用 y 轴量级当照妖镜。"

**1860 版**

> "Grafana 我两条腿走路：自建面板练 PromQL 内功，再导入社区 1860 看板开箱即用——我清楚 1860 靠标准 node_* 指标驱动、个别面板 No data 是采集器没配而非故障，还能用 Uptime 面板反推宿主最近一次开机时间做交叉验证。"

**EulerCopilot 版**

> "openEuler 的 AI 原生路线我有关注：EulerCopilot 用 OS 领域模型 + RAG 把命令交付升级成自然语言运维，智能诊断走巡检→定界→定位三步，调优底座是 A-Tune，微服务跑在 K3s 上——它使能的 iSulad 我亲手练过，这套架构每个部件我都能对上号。"

（iSula 版 / 监控版金句见 day14 笔记与对应手册节）
