# OpenClaw 笔记

[OpenClaw](https://github.com/openclaw/openclaw) 是一个 CLI agent + 多通道机器人框架（飞书、Telegram、Discord、Slack 等）。这里记录我自己的部署方式、踩坑记录和飞书机器人接入指南。

## 目录

- [feishu.md](./feishu.md) — 从零接入飞书机器人（创建应用、配置 OpenClaw、多机器人、调试）
- [llm-backend.md](./llm-backend.md) — 切换 LLM 后端（LiteLLM / OpenAI 兼容代理 / 内网模型网关）
- [examples/openclaw.example.json](./examples/openclaw.example.json) — 配置文件示例（脱敏）

## 全局速查

```bash
# 安装（npm 全局）
sudo npm i -g openclaw@latest

# 初始化配置（交互）
openclaw onboard
openclaw configure

# 启动 gateway（loopback，默认端口 18789）
nohup openclaw gateway run --bind loopback --port 18789 --force \
  > /tmp/openclaw-gateway.log 2>&1 &

# 查看通道状态
openclaw channels status --probe

# 看日志
tail -f /tmp/openclaw-gateway.log
```

## 配置文件位置

- 主配置：`~/.openclaw/openclaw.json`
- 会话/凭证：`~/.openclaw/credentials/`、`~/.openclaw/sessions/`
- 插件运行时：`~/.openclaw/plugin-runtime-deps/`

修改主配置后必须**重启 gateway** 才会生效。
