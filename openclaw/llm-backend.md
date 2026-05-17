# 切换 LLM 后端

OpenClaw 所有通道（飞书、Telegram、CLI agent…）共用 `agents.defaults.model.primary` 指向的模型。改一次，全部生效。

## 1. 模型 ID 命名

格式：`<provider>/<modelId>`，例如：

- `litellm/gpt-5.4`
- `litellm/claude-opus-4.7-1m-internal`
- `openai/gpt-4o-mini`

`<provider>` 必须在 `models.providers` 里注册过。

## 2. 自定义 OpenAI 兼容后端（LiteLLM / 内网网关）

任何对外暴露 `/v1/chat/completions`、`/v1/models` 的服务都能接。比如本地 LiteLLM 或公司内网模型网关。

### 2.1 注册 provider

`~/.openclaw/openclaw.json`：

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "litellm": {
        "baseUrl": "http://YOUR_GATEWAY_HOST:PORT",
        "api": "openai-completions",
        "models": [
          {
            "id": "claude-opus-4.7-1m-internal",
            "name": "Claude Opus 4.7 (1M context)",
            "reasoning": true,
            "input": ["text", "image"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 1000000,
            "maxTokens": 64000
          }
        ]
      }
    }
  },
  "auth": {
    "profiles": {
      "litellm:default": {
        "provider": "litellm",
        "mode": "api_key"
      }
    }
  }
}
```

> `cost` 全 0 表示不参与计费统计；真实付费端点写实际单价。
> 如果后端需要 API key，用 `openclaw channels login` 或环境变量注入（避免明文写 JSON）。

### 2.2 指定为主模型

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "litellm/claude-opus-4.7-1m-internal"
      },
      "models": {
        "litellm/claude-opus-4.7-1m-internal": { "alias": "LiteLLM" }
      }
    }
  }
}
```

### 2.3 验证端点

先用 curl 直连确认端点能返回 200：

```bash
curl -s http://YOUR_GATEWAY_HOST:PORT/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-opus-4.7-1m-internal",
    "messages": [{"role":"user","content":"ping"}],
    "max_tokens": 20,
    "stream": false
  }' | head -c 500
```

返回里有 `choices[0].message.content` 就 OK。

### 2.4 重启 gateway

```bash
kill $(pgrep -f openclaw-gateway) 2>/dev/null
nohup openclaw gateway run --bind loopback --port 18789 --force \
  > /tmp/openclaw-gateway.log 2>&1 &
```

## 3. 切回官方 provider

把 `agents.defaults.model.primary` 改成 `openai/gpt-4o` 或 `anthropic/claude-opus-4-6` 之类，重启即可。前提是对应 provider 已经在 `models.providers` 注册并配好 key。

## 4. 调试

```bash
# 看 gateway 把请求打到哪里、什么错
tail -f /tmp/openclaw-gateway.log | grep -iE 'model|completion|provider|error'

# 列出 OpenClaw 认到的模型
openclaw config show 2>/dev/null | grep -A2 primary
```
