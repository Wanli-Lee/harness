# 飞书机器人接入 OpenClaw

把一个飞书自建应用（机器人）接到本地 OpenClaw gateway，让群里 @ 机器人就能跟你配的 LLM 对话。

> 适用版本：OpenClaw ≥ `2026.4`
> 模式：**WebSocket 长连**（最省事，不用配回调 URL、不用公网）

---

## 1. 在飞书开放平台创建应用

打开 https://open.feishu.cn/app → **创建企业自建应用**。

填好应用名（比如「里神本神」）、图标、描述之后，进入应用详情页。

### 1.1 拿凭证

「凭证与基础信息」页 → 记下：

- **App ID**：`cli_xxxxxxxxxxxxxxxx`
- **App Secret**：32 位字符串

> ⚠️ App Secret 等同密码，别提交到 Git，别贴聊天。

### 1.2 启用机器人能力

「应用功能」→「机器人」→ 启用。

### 1.3 开权限

「权限管理」→ 至少开这几个（消息相关）：

- `im:message`
- `im:message:send_as_bot`
- `im:chat`
- `im:resource`（要发图/文件就开）

需要建群、查群成员、调多维表格等再按需加。

### 1.4 事件订阅

「事件与回调」→ 订阅事件：

- `im.message.receive_v1`（接收消息，必开）

**连接方式选「长连接（WebSocket）」**，不要选 HTTP 回调。这样不需要你的机器有公网 IP，OpenClaw 直接对飞书 open API 主动建连。

### 1.5 发版 + 拉群

1. 「版本管理与发布」→ 创建版本 → 提交审核（企业管理员审批）
2. 审批通过后，把机器人加进你要用的飞书群（群设置 → 群机器人 → 添加）

---

## 2. 配置 OpenClaw

编辑 `~/.openclaw/openclaw.json`，在 `channels.feishu` 里加账号。

完整示例（占位符已脱敏，对照 [`examples/openclaw.example.json`](./examples/openclaw.example.json)）：

```json
{
  "channels": {
    "feishu": {
      "enabled": true,
      "dmPolicy": "pairing",
      "connectionMode": "websocket",
      "groupPolicy": "allowlist",
      "requireMention": true,
      "accounts": {
        "default": {
          "appId": "cli_XXXXXXXXXXXXXXXX",
          "appSecret": "REPLACE_WITH_APP_SECRET"
        },
        "robot2": {
          "appId": "cli_YYYYYYYYYYYYYYYY",
          "appSecret": "REPLACE_WITH_APP_SECRET"
        }
      }
    }
  }
}
```

字段说明：

| 字段 | 取值 | 说明 |
|---|---|---|
| `enabled` | `true` | 开启飞书通道 |
| `connectionMode` | `websocket` | 跟 1.4 选的方式对应 |
| `groupPolicy` | `allowlist` / `all` | 群消息策略；allowlist 只响应白名单群 |
| `requireMention` | `true` | 群里必须 @机器人才会回（强烈建议开） |
| `dmPolicy` | `pairing` / `open` / `closed` | 私聊策略；pairing 走配对码 |
| `accounts.<key>` | 自定义 | 一个机器人一个 key，可以多个 |
| `accounts.<key>.appId` | 飞书 App ID | 1.1 拿到的 |
| `accounts.<key>.appSecret` | 飞书 App Secret | 1.1 拿到的 |

> `<key>` 是 OpenClaw 内部用的标识（用于 `--account` 选择、日志区分），可以是任意字符串。**飞书后台里的应用显示名跟它无关**。

---

## 3. 重启 gateway 并验证

```bash
# 停掉所有现有 gateway
kill $(pgrep -f openclaw-gateway) 2>/dev/null

# 启动新的
nohup openclaw gateway run --bind loopback --port 18789 --force \
  > /tmp/openclaw-gateway.log 2>&1 &

sleep 5

# 检查
openclaw channels status --probe
```

期望输出：

```
Gateway reachable.
- Feishu default: enabled, configured, running, works
- Feishu robot2:  enabled, configured, running, works
```

`works` 才算成功。如果是 `probe failed`：

- App Secret 抄错？
- 应用版本没发布或被驳回？
- 权限里 `im:message` 漏开？
- 看日志：`tail -f /tmp/openclaw-gateway.log`

---

## 4. 在飞书里实测

群里 @机器人 + 一句话，比如：

```
@里神本神 你好，你现在用的什么模型？
```

机器人会调本地配的主模型回复。模型怎么配见 [`llm-backend.md`](./llm-backend.md)。

### 私聊配对

如果允许私聊（`dmPolicy: pairing`），第一次私聊机器人会要配对码。在终端跑：

```bash
openclaw channels directory --channel feishu
```

按提示拿到 open_id / 配对码，发给机器人即可。

---

## 5. 用接口确认机器人身份

如果你忘了某个 `accounts.<key>` 对应飞书后台哪个应用，用 App ID + Secret 换 token 然后查 bot info：

```bash
APPID=cli_XXXXXXXXXXXXXXXX
SECRET=REPLACE_WITH_APP_SECRET

TOKEN=$(curl -s -X POST \
  https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal \
  -H "Content-Type: application/json" \
  -d "{\"app_id\":\"$APPID\",\"app_secret\":\"$SECRET\"}" \
  | python3 -c "import json,sys;print(json.load(sys.stdin)['tenant_access_token'])")

curl -s https://open.feishu.cn/open-apis/bot/v3/info \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

返回里 `bot.app_name` 就是飞书后台显示名。

---

## 6. 添加第 N 个机器人

1. 让对方在飞书开放平台用自己账号创应用（步骤同 §1）
2. 拿到 App ID + Secret 后，在 `accounts` 下追加一项：

   ```json
   "colleague1": {
     "appId": "cli_ZZZZZZZZZZZZZZZZ",
     "appSecret": "REPLACE_WITH_APP_SECRET"
   }
   ```

3. 重启 gateway → `channels status --probe` 看到 `colleague1: works`

> 注意：所有机器人共享同一个 gateway 进程、同一台机器、同一份 LLM 配置和配额。如果对方要完全独立的实例，应该让他在自己机器上独立部署 OpenClaw。

---

## 7. 常见坑

| 现象 | 排查 |
|---|---|
| `probe failed` | App Secret、权限、版本发布、网络（公司代理常见） |
| 群里 @ 没反应 | `requireMention` 是否开了？机器人是否真的在群里？是否被群管理员限制 |
| 私聊不回 | `dmPolicy` 设置；私聊默认 pairing |
| 重启后旧实例还在 | `pgrep -f openclaw-gateway` 再 `kill`；注意有时候有多个残留 |
| 改了 JSON 没生效 | gateway 是常驻进程，**必须重启** |
| 模型不对 | 改的是 `agents.defaults.model.primary`，见 `llm-backend.md` |

---

## 参考

- OpenClaw 官方文档：https://docs.openclaw.ai
- 飞书开放平台：https://open.feishu.cn/document
- 仓库 README：[../README.md](../README.md)
