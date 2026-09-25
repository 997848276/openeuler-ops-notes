# DAY15 · openEuler Copilot System（EulerCopilot）理解向

> 定位：openEuler 社区的「运维版 Copilot」，把操作系统从「命令交付」进化成「自然语言交互」。
> 说明：本篇为**理解向**笔记。EulerCopilot 推荐配置 16 核 / 64GB 内存 / 可选 GPU（V100×4），
> 本地 2 核 3G 的 VM 带不动，故**只做定位与能力理解，不安装不跑**。无 GPU 也可走 OpenAI 接口接入。

## 一、一句话定位

基于 **ChatGLM 训练的 OS 领域模型** + **RAG（检索增强）**，让你用人话问、它规划任务、调工具、给答案。
它不是「装个 rpm 就有」，而是一套**部署在服务端、外接大模型 API** 的智能问答系统。

## 二、能做什么（三入口 + 四助理 + 两杀手锏）

- **三入口**：Web 网页问答 / 智能 Shell（终端里说人话做运维）/ IDE 插件
- **四助理**：问答助理、操作助理、巡检助理、编程助理
- **杀手锏① 智能诊断三步**：
  1. 巡检（Inspection）：发现异常 IP + 关联容器 ID + 异常指标（CPU/内存）
  2. 定界（Demarcation）：从巡检结果里锁定 Top3 根因指标
  3. 定位（Detection）：对根因做 profiling 下钻，给出栈/系统时间/热点指标
  - 👉 与你这两周手练的「排障链」逻辑一致，只是它自动化了
- **杀手锏② 智能调优**：采集 CPU/IO/网络/应用指标 → 生成分析报告 → **一键执行调优脚本**（底座就是 A-Tune）

## 三、架构骨架（微服务 + RAG + K3s）

| 组件 | 端口 | 作用 |
|---|---|---|
| euler-copilot-framework | 8002（内部） | 智能体框架服务 |
| euler-copilot-web | 8080 | 前端界面 |
| euler-copilot-rag | 8005/9988（内部） | 检索增强服务 |
| euler-copilot-vectorize-agent | 8001（内部） | 文本向量化 |
| mysql | 3306 | 业务库 |
| redis | 6379 | 缓存 |
| postgres | 5432 | 向量数据库 |
| secret_inject | 无 | 配置安全复制 |

- 运行底座：**K3s + Helm**（轻量 K8s，正是在练的东西）
- 知识层：**RAG** = 知识索引 → 多通道召回（向量+关键词+Chat2DB）→ 重排 → 收敛（和 WorkBuddy 用的是同款逻辑）

## 四、与已学知识串线（重点）

- 微服务跑在 **K3s** 上 → K8s 练习 15~17 你练过
- 组件里有 **MySQL / Redis / Postgres** → 数据库 DB-1~4 练过
- 它使能的生产力工具列表里有 **iSulad** → DAY11-12 你亲手玩过！
- **RAG 检索增强** → 你现在用的 WorkBuddy 同款

## 五、坑位（必记）

1. **名字两副面孔**：社区文档有时叫「openEuler Copilot System」，新版叫「openEuler Intelligence」，是一套东西
2. **不是单机工具**：必须 K3s + 向量库 + 大模型 API，本地小 VM 只能理解不能部署
3. **RAG 才是灵魂**：问答质量靠语料治理（片段关系提取/OCR/衍生摘要），模型本身只是底座

## 面试一句话

> "openEuler 的 AI 原生路线我有关注：EulerCopilot 用 OS 领域模型 + RAG 把命令交付升级成自然语言运维，智能诊断走巡检→定界→定位三步，调优底座是 A-Tune，微服务跑在 K3s 上——它使能的 iSulad 我亲手练过，这套架构每个部件我都能对上号。"
