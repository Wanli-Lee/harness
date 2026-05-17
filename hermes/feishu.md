# Hermes 接入飞书

把 Hermes Agent 接入飞书机器人，跟 OpenClaw 一样走 **WebSocket 长连**（不需要公网回调）。

> 测试版本：Hermes Agent `v0.14.0` / 2026.5.16

---

## 1. 飞书后台准备

跟 [`../openclaw/feishu.md`](../openclaw/feishu.md) §1 完全一样：

1. https://open.feishu.cn/app → 创建企业自建应用
2. 「凭证与基础信息」拿 **App ID** + **App Secret**
3. 「应用功能」启用 **机器人**
4. 「权限管理」开 `im:message` / `im:message:send_as_bot` / `im:chat` / `im:resource`
5. 「事件与回调」订阅 `im.message.receive_v1`，**连接方式选「长连接 (WebSocket)」**
6. 「版本管理与发布」发布 + 审批通过
7. 把机器人拉进群

> 同一个飞书 App **不能**被两个进程同时持有 WebSocket 连接。如果你已经在 OpenClaw 那边用了它，要么停 OpenClaw，要么给 Hermes 单建一个 App。详见 [`coexist-with-openclaw.md`](./coexist-with-openclaw.md)。

## 2. 写 `~/.hermes/.env`

只要这几行：

```bash
FEISHU_APP_ID=cli_XXXXXXXXXXXXXXXX
FEISHU_APP_SECRET=REPLACE_WITH_APP_SECRET
FEISHU_DOMAIN=feishu               # 国际版 Lark 用 lark
FEISHU_CONNECTION_MODE=websocket

# 单用户/实验环境：放开所有人；正式部署务必改成 allowlist
GATEWAY_ALLOW_ALL_USERS=true
# FEISHU_ALLOWED_USERS=ou_xxx,ou_yyy
# FEISHU_HOME_CHANNEL=oc_xxxxxxxxxxxxxxxxxxxxxxxx
```

完整模板见 [`examples/env.example`](./examples/env.example)。

### 字段速查

| 变量 | 说明 |
|---|---|
| `FEISHU_APP_ID` / `FEISHU_APP_SECRET` | 飞书后台凭证 |
| `FEISHU_DOMAIN` | `feishu`（国内）/ `lark`（海外） |
| `FEISHU_CONNECTION_MODE` | `websocket`（推荐）/ `webhook` |
| `FEISHU_ALLOWED_USERS` | 逗号分隔 open_id；只允许这些人 DM |
| `FEISHU_HOME_CHANNEL` | cron 投递的默认聊天 id（`oc_…`） |
| `GATEWAY_ALLOW_ALL_USERS` | `true` 时跳过所有平台 allowlist（**只在 lab 用**） |

## 3. 启动 gateway

前台（看日志方便）：

```bash
hermes gateway run
```

后台（detached，注意要清空代理 env，飞书在国内，走 SOCKS/HTTP 代理反而连不上）：

```bash
PIDS=$(pgrep -f "hermes gateway")
for p in $PIDS; do kill -9 "$p"; done
setsid env -i HOME=$HOME PATH=$PATH USER=$USER LANG=C.UTF-8 \
  bash -c "hermes gateway run </dev/null > /tmp/hermes-gateway.log 2>&1" \
  </dev/null >/dev/null 2>&1 &
sleep 10
hermes gateway status
tail -20 /tmp/hermes-gateway.log
```

启动成功长这样：

```
[Lark] [INFO] connected to wss://msg-frontier.feishu.cn/ws/v2?... [conn_id=...]
```

`connected` 出来就 OK，群里 @机器人 试一下即可。

## 4. 安装为系统服务（可选）

```bash
hermes gateway install            # 当前用户 systemd --user
sudo hermes gateway install --system   # 系统级
hermes gateway start
hermes gateway stop
hermes gateway restart
```

## 5. 常见坑

| 现象 | 排查 |
|---|---|
| `connect failed, err: connecting through a SOCKS proxy requires python-socks` | 你 shell 里有 `ALL_PROXY=socks5://...`。**清空代理 env 再启动 gateway**（飞书是国内域名） |
| Lark 一直 reconnect | App 没发版？审批没过？App Secret 错？|
| 群里 @ 没反应 | App 没拉进群；或 `GATEWAY_ALLOW_ALL_USERS` 没开且不在 `FEISHU_ALLOWED_USERS` |
| 模型不响应 | `hermes -z "ping"` 单测；看 `~/.hermes/.env` 里 provider/key 是否对 |
| 改了 .env 没生效 | gateway 必须重启 |

## 6. 验证机器人身份

跟 OpenClaw 那篇一样：

```bash
APPID=cli_XXXXXXXXXXXXXXXX
SECRET=REPLACE_WITH_APP_SECRET
TOKEN=$(curl -s -X POST https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal \
  -H "Content-Type: application/json" \
  -d "{\"app_id\":\"$APPID\",\"app_secret\":\"$SECRET\"}" \
  | python3 -c "import json,sys;print(json.load(sys.stdin)['tenant_access_token'])")
curl -s https://open.feishu.cn/open-apis/bot/v3/info \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

返回里 `bot.app_name` 就是飞书后台机器人显示名。

## 参考

- Hermes 官方文档：https://hermes-agent.nousresearch.com/docs/user-guide/messaging/feishu
- 飞书开放平台：https://open.feishu.cn/document
