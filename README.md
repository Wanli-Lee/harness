# harness

个人运维/集成笔记仓库。每个子目录对应一个工具或服务，里面是落地指南、配置示例、踩坑记录。

## 目录

| 子目录 | 内容 |
|---|---|
| [`openclaw/`](./openclaw) | OpenClaw（CLI agent + 多通道机器人）部署、飞书机器人接入、LLM 后端切换等 |
| [`hermes/`](./hermes) | Hermes Agent（self-improving，飞书/TG/Slack 等）安装、飞书接入、LLM 后端、与 OpenClaw 共存 |

## 约定

- 不提交任何真实 secret（token、appSecret、内网 IP、手机号等），全部用占位符。
- 配置示例统一放在每个子目录的 `examples/` 下。
- 中文笔记为主，命令/字段保留英文。
