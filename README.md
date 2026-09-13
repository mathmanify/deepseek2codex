# 把 OpenCode Go 的 DeepSeek V4.1 Flash 接进 Codex：App 与 CLI 两种姿势

> 实测环境：macOS + ChatGPT.app（Codex App）+ codex-cli 0.153/0.154，2026-09-13 全程验证通过。
> 本文自包含：两种接入方式、完整配置、模型目录生成脚本、验证命令与已知坑。

## TL;DR

- OpenCode Go（$10/月）官方允许「任何 agent」，**Codex 在官方 Validated Clients 名单里**（Go 识别 Codex 原生 session header）。
- Codex 0.153+ 的自定义 provider **只支持 `wire_api = "responses"`**；Go 的 `https://opencode.ai/zen/go/v1/responses` 实测支持 `deepseek-v4.1-flash`，并且接受 `reasoning.effort = "max"`。
- **方式一（全局）**：App 与 CLI 都用 DeepSeek；代价是 App 里 ChatGPT 的 GPT 额度/余额显示消失（原因见 §4）。
- **方式二（CLI 专属）**：App 保持 ChatGPT 原样、余额照常显示，CLI 用 `codex --profile opencode-go` 走 DeepSeek。

---

## 0. 前提

1. OpenCode Go 订阅与 API Key：在 [opencode.ai/auth](https://opencode.ai/auth) 订阅后复制（Zen 与 Go 共用同一个 key）。
2. 已安装 Codex CLI 或 ChatGPT 桌面版，且至少启动过一次（生成 `~/.codex`）。
3. 如果你想先看官方依据：Go 文档的 "Where can I use it?" 明确列出 Codex 为已验证客户端；Codex 配置参考确认 `wire_api` 只支持 `responses`、`model_catalog_json` 可选。

## 1. 准备密钥（两种方式通用）

```bash
mkdir -p ~/.secrets && chmod 700 ~/.secrets
printf '%s' 'sk-你的OpenCode密钥' > ~/.secrets/opencode-go.key
chmod 600 ~/.secrets/opencode-go.key
```

为什么用「文件 + auth 命令」而不是 `env_key`：macOS 的 GUI App 不继承 shell 环境变量，命令式取 token 对 App 和 CLI 都生效，且密钥不会进入配置文件正文。

## 2. 注册 provider（两种方式通用，追加到 `~/.codex/config.toml` 末尾）

```toml
[model_providers.opencode-go]
name = "OpenCode Go"
base_url = "https://opencode.ai/zen/go/v1"
wire_api = "responses"

[model_providers.opencode-go.auth]
command = "/bin/cat"
args = ["/Users/你的用户名/.secrets/opencode-go.key"]
```

## 3. 生成模型目录 JSON（推荐，两种方式通用）

DeepSeek 不在 Codex 原生目录里。自建目录 JSON 有两个作用：让它出现在 `/model` 选择器、并声明 1M 上下文与推理档位。

**形状必须是 `{"models":[...]}`**（裸数组会报 `invalid type: map, expected a sequence`，实测踩过）。下面的脚本克隆本机原生条目再改字段，实测通过：

```python
# python3 gen_catalog.py
import json, os

home = os.path.expanduser("~/.codex")
cache = json.load(open(f"{home}/models_cache.json"))   # 不存在时先随便启动一次 Codex
models = cache["models"]
tpl = next(m for m in models if m["slug"] == "gpt-5.6-luna")  # 任意原生条目做模板

e = json.loads(json.dumps(tpl))
e.update({
    "slug": "deepseek-v4.1-flash",
    "display_name": "DeepSeek-V4.1-Flash",
    "description": "DeepSeek V4.1 Flash via OpenCode Go",
    "context_window": 1000000,
    "max_context_window": 1000000,
    "effective_context_window_percent": 95,
    "auto_compact_token_limit": 900000,
    "default_reasoning_level": "max",
    "supported_reasoning_levels": [
        {"effort": "low",  "description": "Fast responses with lighter reasoning"},
        {"effort": "high", "description": "Extra high reasoning depth for complex problems"},
        {"effort": "max",  "description": "Maximum reasoning depth for the hardest problems"},
    ],
    "priority": 1,
    "visibility": "list",
})

out = [m for m in models if m.get("slug") != e["slug"]] + [e]
json.dump({"models": out}, open(f"{home}/opencode-go-models.json", "w"), ensure_ascii=False, indent=2)
print("wrote", f"{home}/opencode-go-models.json")
```

## 4. 方式一：让 Codex App 也用 DeepSeek（全局切换）

在 `~/.codex/config.toml` 的**顶层键区**（第一个 `[table]` 之前）加入/修改这些键：

```toml
model = "deepseek-v4.1-flash"
model_provider = "opencode-go"
model_reasoning_effort = "max"
model_context_window = 1000000
model_auto_compact_token_limit = 900000
model_catalog_json = "/Users/你的用户名/.codex/opencode-go-models.json"
```

然后完全退出并重开 ChatGPT.app（App 只在启动时读配置）。

**代价（重要）**：GPT 模型的额度/余额显示会消失。原因不是账号问题：Codex 只在当前 provider 声明 `requires_openai_auth = true` 时才拉取本地 ChatGPT 账号的限额（上游 issue #34134）；而 Go 直连不能加这个标志——官方文档禁止它与 `auth` 命令组合，且那会把 ChatGPT 凭据发给 Go，必然认证失败。

想切回原生：删掉 `model_provider` 行、`model` 改回 `gpt-5.6-sol`，重启 App。

## 5. 方式二：只让 Codex CLI 用 DeepSeek（App 保持原样）

基础 `~/.codex/config.toml` 保持 ChatGPT 原生（`model = "gpt-5.6-sol"`，不要 `model_provider` / `model_catalog_json`；§2 的 provider 块保留即可）。

新建 `~/.codex/opencode-go.config.toml`（profile 文件，用顶层键）：

```toml
model = "deepseek-v4.1-flash"
model_provider = "opencode-go"
model_reasoning_effort = "max"
model_context_window = 1000000
model_auto_compact_token_limit = 900000
model_catalog_json = "/Users/你的用户名/.codex/opencode-go-models.json"
```

用法：

```bash
codex --profile opencode-go           # TUI：状态栏显示 deepseek-v4.1-flash max
codex exec --profile opencode-go "…"  # 非交互
```

实测结果：`/model` 选择器里 `deepseek-v4.1-flash (current)` 直接可见；App 不受影响，GPT 余额照常显示。

两个实用细节：

- 想让终端里 `codex` 默认就是 DeepSeek：`alias codex='codex --profile opencode-go'`（只影响交互终端；要 GPT 时用 `command codex`）。
- `codex app` **不能**带 `--profile`（0.153 实测报 `unexpected argument`）；它本质是 `open -a` + 深链启动器，App 永远只读基础 `config.toml`。所以 App 想用 DeepSeek 只能用方式一。

## 6. 验证与排错

```bash
# 1) Key 与端点
curl -s https://opencode.ai/zen/go/v1/models \
  -H "Authorization: Bearer $(cat ~/.secrets/opencode-go.key)" | grep deepseek-v4.1-flash

# 2) 直连 Responses + max 推理
curl -s -X POST https://opencode.ai/zen/go/v1/responses \
  -H "Authorization: Bearer $(cat ~/.secrets/opencode-go.key)" \
  -H "Content-Type: application/json" \
  -d '{"model":"deepseek-v4.1-flash","input":"Say OK","stream":false,"reasoning":{"effort":"max"}}'

# 3) Codex 侧
codex debug models | grep deepseek-v4.1-flash
codex exec --profile opencode-go --json "reply only with: ok"
```

常见坑：

- 目录 JSON 报 `invalid type: map, expected a sequence` → 顶层必须是 `{"models":[...]}`。
- 别写 `wire_api = "chat"`：0.153 起只支持 `responses`（社区方案里"404 就改 chat"的旧建议已失效）。
- Go 的 **free 模型**（`-free`）只走 chat 端点，Codex 用不了；付费模型（`zen/go/v1`）才通。
- Anthropic `/v1/messages` 端点模型（Qwen Max、MiniMax M 系）直连 Codex 不可行，需要 opencodex 这类 Responses↔Chat/Messages 翻译代理。
- 长会话报 400：把 `model_auto_compact_token_limit` 降到 `400000`（顶层和目录 JSON 同步改）。
- 推理档位：`ultra` 只有 GPT-5.6 Sol/Terra 有；DeepSeek 用 `max`（目录里声明 low/high/max）。

## 7. 参考

- OpenCode Go 文档（端点、Validated Clients、用量）：https://opencode.ai/docs/go
- Codex 配置参考（wire_api / context / reasoning / catalog）：https://learn.chatgpt.com/docs/config-file/config-reference
- 额度显示门控的上游 issue：https://github.com/openai/codex/issues/34134
- 社区实践：BitQAI/codex-opencode-setup、ZenoZ-Liu/codex-opencode-go、https://opencodex.me
