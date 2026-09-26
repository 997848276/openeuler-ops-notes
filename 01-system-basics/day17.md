# DAY13 · EulerCopilot CLI 接 SiliconFlow 实操（2026-09-26，128 / oe-base）

> 配套三件套手册：`openEuler命令解读与类比.md` 第 22 节。本文件是当天实操流水，推仓库时改名为 `day13.md` 落 `01-system-basics/`。

## 0. 背景与目标

openEuler 智能化入口（day15 理解向）要落一入口。官方 Copilot System 全栈需 K3s+Helm+MySQL/Redis/Postgres 向量库+Ollama 大模型，硬件要求 32核/64GB/500GB/GPU → 128 只有 2C/3.3Gi，跑不动。收敛到 **eulercopilot-cli** 命令行助理。官方默认后端 `eulercopilot.gitee.com` 已 404（仓库迁移 AtomGit，内测 Key 服务随迁下线）→ 用 config.json 自定义 model 字段接 SiliconFlow（OpenAI 兼容）。

## 1. 侦察（全绿）

```bash
cat /etc/openEuler-release          # openEuler release 24.03 (LTS)
sudo dnf config-manager --add-repo https://repo.oepkgs.net/openeuler/rpm/$(sed 's/release //;s/[()]//g;s/ /-/g' /etc/openEuler-release)/extras/$(uname -m)
sudo dnf makecache                  # metadata 46MB，源生效
sudo dnf list eulercopilot-cli      # 1.2.1-5 在源里
curl -sI https://api.siliconflow.cn/v1/chat/completions   # 出网通
nproc; free -h; df -h               # 2C / 3.3Gi / 64G 用 10%
```

## 2. 安装（GPG 坑）+ 初始化

```bash
sudo dnf install -y --nogpgcheck eulercopilot-cli   # 社区包未签名 → GPG check FAILED → --nogpgcheck
copilot --help                                  # 四模式 chat/plugin/diagnose/tuning
copilot --init                                  # 交互问 Key，生成 ~/.config/eulercopilot/config.json
```

解读：`--nogpgcheck` 是信任决策（官方源禁用）；config.json 运行时生成（`rpm -ql` 查不到）；init 不强校验（假 Key 也写入）。

## 3. 决战三连（本案精华）

**① backend 名逆向（strings）**

```bash
strings /usr/bin/copilot | grep -iE 'model_|framework_|backend|spark_' | sort -u
# → copilot.backends.framework_api) / llm_service) / openai_api) / spark_api)
```

copilot 是 PyInstaller 打包的 Python 程序，合法 backend 只有 `framework/llm_service/openai/spark`。填 `"model"` → “未正确配置 LLM 后端”（本地校验，不出网）。`model_*` 三字段归 `openai_api` 消费。

**② 切 openai + curl 隔离定位 URL**

```bash
sed -i 's/"backend": "model"/"backend": "openai"/' ~/.config/eulercopilot/config.json
SK="sk-你的Key"
curl -s https://api.siliconflow.cn/v1/chat/completions \
  -H "Authorization: Bearer $SK" -H "Content-Type: application/json" \
  -d '{"model":"tencent/Hy4-preview","messages":[{"role":"user","content":"hi"}],"max_tokens":16}'
```

curl 成功 → 网络/endpoint/Key/模型名四要素全对。CLI 不自动补路径 → config 的 `model_url` 必须写全 `https://api.siliconflow.cn/v1/chat/completions`。

**③ 推理模型崩溃 → 换非推理模型（法医推理）**

```bash
copilot "用一句话介绍什么是LVM"   # Traceback: _query_llm_service → _stream_response:106
                                    # TypeError: can only concatenate str (not "NoneType") to str
sed -i 's#"model_name": "tencent/Hy4-preview"#"model_name": "deepseek-ai/DeepSeek-V3"#' ~/.config/eulercopilot/config.json
copilot "用一句话介绍什么是LVM"   # → "LVM（Logical Volume Manager）是 Linux 下的逻辑卷管理器……" 🎉
```

崩溃点在流式解析 = 已收到 200 在拼回答时炸。Hy4-preview 是推理模型，回答带 `reasoning_content`，流式块大量为 None → 内测版解析器 `str+None` 崩。换 DeepSeek-V3（非推理）一次通关。

## 4. 最终可用 config

```json
{
  "backend": "openai",
  "model_url": "https://api.siliconflow.cn/v1/chat/completions",
  "model_api_key": "sk-xxxx（已在截图裸露，建议后台重置）",
  "model_name": "deepseek-ai/DeepSeek-V3"
}
```

用法：`copilot "问题"` 单发 / `copilot` 进交互 REPL（`exit` 或 Ctrl+C 退出）。备份：`config.json.bak`（旧版）。

## 5. 坑位速查（DAY13 六连）

| # | 坑 | 正解 |
|---|---|---|
| 1 | OEPKGS 包 GPG check FAILED | `--nogpgcheck`（社区源信任决策，官方源禁用） |
| 2 | init 只配 Gitee，已 404 | init 随便填，事后手改 config.json 的 model_* |
| 3 | backend 填 "model" → 未正确配置 | 合法值 framework/llm_service/openai/spark，用 **openai** |
| 4 | model_url 写 base /v1 → REPL 内 404 | 写全路径 /v1/chat/completions |
| 5 | 推理模型(Hy4-preview) → str+None Traceback | 换非推理模型 DeepSeek-V3 |
| 6 | cat /usr/bin/copilot 灌 2.39MB 二进制卡死终端 | cat 前先 file；ELF 用 strings；救场=断开重连 |

## 6. 面试一句话

> “我把 EulerCopilot CLI 接到了自选国产大模型 API：官方服务下线后，用 strings 逆向出四个合法 backend 名、用 curl 把 API 通路和 CLI 行为解耦定位出它不自动拼接 endpoint、再从流式解析的 Traceback 反推出它不兼容推理模型的 reasoning_content 字段，换非推理模型通关。**黑盒排障三件套：逆向看它读什么、隔离测每段通路、从崩溃点反推根因**。”

## 7. 收官账

- ✅ eulercopilot-cli 1.2.1（OEPKGS 装机）+ SiliconFlow（DeepSeek-V3）+ `copilot` 问答闭环
- ✅ 方法论：strings 逆向 / curl 隔离 / 崩溃点反推 / 止损线纪律
- 留存：config.json（openai+全路径+DeepSeek-V3）、config.json.bak
- 遗留：①Hy4-preview 崩溃属 CLI 内测 bug，关注版本更新 ②Key 建议重置 ③本文件推仓库改名 day13.md
