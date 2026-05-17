# Hermes 与 OpenClaw 同机共存

两套 agent 框架可以在同一台机器同时跑，但有几个雷区。

## 1. 飞书 App 不能共用

同一个飞书 App ID 只能有**一个**进程持有 WebSocket 长连接。如果两边都用同一个 App ID：

- 后启动那个会把先连的踢掉，循环抢连
- 飞书侧可能直接限流

**做法**：为 Hermes 单建一个飞书 App（比如显示名「Hermes」），跟 OpenClaw 的「里神 / 里神本神」是不同的 App ID。群里你就有两个机器人。

## 2. 端口不冲突

- OpenClaw gateway：`127.0.0.1:18789`（控制面 HTTP）
- Hermes gateway：默认不开 HTTP 端口（websocket-only 模式）
  - 如果开了 webhook 模式，默认 `127.0.0.1:8765`

不撞。

## 3. 代理（proxy）

OpenClaw gateway 不在乎 `ALL_PROXY`；Hermes 用的 Lark Python SDK 会读 `ALL_PROXY` / `HTTPS_PROXY`，遇到 `socks5://...` 但没装 `python-socks` 就报错。

启动 Hermes gateway **务必把 SOCKS 代理 env 清掉**（飞书在国内，不需要走代理）：

```bash
setsid env -i HOME=$HOME PATH=$PATH USER=$USER LANG=C.UTF-8 \
  bash -c "hermes gateway run </dev/null > /tmp/hermes-gateway.log 2>&1" \
  </dev/null >/dev/null 2>&1 &
```

或者写 systemd unit 时显式 `Environment=` 清掉 proxy 变量。

## 4. 模型后端可以共享

两个框架都能接同一个 `http://YOUR_GATEWAY_HOST:PORT/v1`：

- OpenClaw：`models.providers.litellm.baseUrl`，详见 [`../openclaw/llm-backend.md`](../openclaw/llm-backend.md)
- Hermes：`model.provider=custom` + `model.base_url`，详见 [`./llm-backend.md`](./llm-backend.md)

但要注意上游有没有 per-key 并发/速率限制；两套 gateway 都开自动 cron / 后台任务时可能撞上限。

## 5. 数据目录隔离

| 框架 | 数据根目录 |
|---|---|
| OpenClaw | `~/.openclaw/` |
| Hermes | `~/.hermes/` |

互不影响。

## 6. 进程管理

```bash
# OpenClaw
pgrep -fa openclaw-gateway
kill $(pgrep -f openclaw-gateway)

# Hermes
hermes gateway status
hermes gateway stop          # 如果装了 systemd 服务
# 否则手动 kill
PIDS=$(pgrep -f "hermes gateway"); for p in $PIDS; do kill -9 "$p"; done
```

## 7. 从 OpenClaw 迁移到 Hermes

```bash
hermes claw migrate
```

会把 OpenClaw 的会话/配置导一份过来。**不会**自动接管 OpenClaw 的飞书 App。如果想完全替换：

1. `hermes claw migrate`
2. 停 OpenClaw gateway
3. 在 OpenClaw 的 `~/.openclaw/openclaw.json` 里把 `channels.feishu.enabled` 改成 `false`（或删 `accounts.<key>` 条目）
4. 把那个飞书 App 的 ID/Secret 配到 Hermes `~/.hermes/.env`
5. 启动 Hermes gateway

## 8. 推荐部署形态

- 探索 / 多机器人玩 → 两套并存，不同飞书 App
- 单一生产用 → 选一个，关另一个
- 想保留 OpenClaw 的某些通道（比如 Telegram、自定义工具）但飞书走 Hermes → 完全可以，OpenClaw 关掉 `channels.feishu`，其它通道照旧
