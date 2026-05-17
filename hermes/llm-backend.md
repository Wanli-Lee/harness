# Hermes 切换 LLM 后端

默认 Hermes 用 OpenRouter / Nous Portal。要换成自定义 OpenAI 兼容端点（例如内网模型网关、本地 LiteLLM、Copilot 内部代理），用 `provider: custom`。

## 1. 设置（命令）

```bash
hermes config set model.provider custom
hermes config set model.base_url http://YOUR_GATEWAY_HOST:PORT/v1
hermes config set model.default custom/<model-id>
```

例如把 Claude Opus 4.7 (1M) 通过内部网关暴露在 `YOUR_GATEWAY_HOST:PORT`：

```bash
hermes config set model.provider custom
hermes config set model.base_url http://YOUR_GATEWAY_HOST:PORT/v1
hermes config set model.default custom/claude-opus-4.7-1m-internal
```

## 2. 设置（直接编辑 `~/.hermes/config.yaml`）

```yaml
model:
  default: "custom/claude-opus-4.7-1m-internal"
  provider: custom
  base_url: "http://YOUR_GATEWAY_HOST:PORT/v1"
```

## 3. API Key

如果端点要 key，写到 `~/.hermes/.env`：

```bash
OPENAI_API_KEY=sk-...
```

> Hermes 的 `custom` provider 复用 `OPENAI_API_KEY` 这个环境变量（即使端点不是 OpenAI 官方）。如果端点不要 key，写个占位符也行：`OPENAI_API_KEY=sk-dummy-not-required`。

## 4. 验证

直连 curl 先确认端点能用：

```bash
curl -s http://YOUR_GATEWAY_HOST:PORT/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-opus-4.7-1m-internal",
    "messages": [{"role":"user","content":"ping"}],
    "max_tokens": 10
  }' | head -c 300
```

然后 hermes 单测：

```bash
hermes -z "reply with just the word: pong"
```

返回 `pong` 就通了。

## 5. 列出可用模型

```bash
hermes config show | grep -A2 Model
hermes model              # 交互式重新选
```

## 6. 多个 provider 共存

`hermes model` 里可以选不同 provider；切换时它会改 `config.yaml`。常用的几个：

| provider | 需要 env |
|---|---|
| `openrouter` | `OPENROUTER_API_KEY` |
| `nous-api` | `NOUS_API_KEY` |
| `anthropic` | `ANTHROPIC_API_KEY` |
| `openai-codex` | `hermes auth` |
| `copilot` | `GITHUB_TOKEN` |
| `custom` | `OPENAI_API_KEY` + `base_url` |
| `lmstudio` | （可选）`LM_API_KEY` |
| `ollama-cloud` | `OLLAMA_API_KEY` |

完整列表见 `~/.hermes/config.yaml` 顶部注释。

## 7. fallback

```bash
hermes fallback           # 配主模型失败时自动切换的备用模型
```

正式部署强烈建议配 fallback，避免单端点挂掉机器人就哑火。
