# Hermes Agent 笔记

[Hermes Agent](https://github.com/NousResearch/hermes-agent) 是 Nous Research 开源的 self-improving AI agent，支持多通道（Telegram / Discord / Slack / Feishu / Signal / WhatsApp / CLI）、持久 skills、跨会话记忆。

## 目录

- [feishu.md](./feishu.md) — 接入飞书机器人（WebSocket 长连）
- [llm-backend.md](./llm-backend.md) — 接 OpenAI 兼容自定义后端（用 Claude Opus 4.7）
- [coexist-with-openclaw.md](./coexist-with-openclaw.md) — Hermes 和 OpenClaw 同机共存的注意点
- [examples/env.example](./examples/env.example) — `~/.hermes/.env` 示例（脱敏）

## 一键安装（Linux/macOS/WSL2）

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh \
  | bash -s -- --skip-setup --skip-browser
```

- `--skip-setup`：跳过交互式向导（手动改 `~/.hermes/.env` 更可控）
- `--skip-browser`：跳过 Playwright/Chromium（省 ~400 MB；用到浏览器工具再装）

完成后：

```bash
source ~/.bashrc
hermes --version
hermes -z "ping"     # 一次性问答（验证模型链路）
hermes gateway run   # 启动消息网关（前台）
```

## 文件位置

| 路径 | 用途 |
|---|---|
| `~/.hermes/config.yaml` | 主配置（模型、TUI 行为） |
| `~/.hermes/.env` | 所有 API key、平台凭证 |
| `~/.hermes/hermes-agent/` | 源码（升级用 `hermes update`） |
| `~/.hermes/sessions/` | 会话历史（SQLite + FTS5） |
| `~/.hermes/skills/` | 学到的 skill 文件 |
| `~/.hermes/memories/` | 跨会话记忆 |
| `~/.hermes/logs/` | gateway 日志 |

## 常用命令

```bash
hermes                   # 进入 TUI
hermes -z "<prompt>"     # 一次性
hermes model             # 交互式换 provider/模型
hermes config show       # 查当前配置
hermes config set <k> <v>
hermes gateway run       # 前台跑网关
hermes gateway status
hermes doctor            # 诊断
hermes claw migrate      # 从 OpenClaw 迁移
```

升级：`hermes update`。
