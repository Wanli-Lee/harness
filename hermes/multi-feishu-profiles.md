# 同机跑多个飞书机器人（Hermes profile）

Hermes 的环境变量是一份 `~/.hermes/.env`，所以**一份配置只能接一个飞书 App**。要在同一台机器上同时跑多个飞书机器人，正确做法是用 **`hermes profile`**：每个 profile 一套 `.env` + `config.yaml` + 自己的 gateway 进程。

> Hermes 不支持单 gateway 多飞书 App。别想绕过 — 同 App ID 多连接会被飞书踢。

## 1. 创建第二个 profile

```bash
hermes profile create lwl --clone
```

- `--clone`：把当前 active profile 的 `config.yaml` + `.env` + `SOUL.md` 复制过来（同模型、同其它平台 token），只需要改飞书账号
- 会生成 wrapper：`/home/USER/.local/bin/lwl`（等价 `hermes --profile lwl ...`）
- 新数据目录：`~/.hermes/profiles/lwl/`

## 2. 改新 profile 的飞书凭证

```bash
sed -i 's|^FEISHU_APP_ID=.*|FEISHU_APP_ID=cli_XXXXXXXXXXXXXXXX|' ~/.hermes/profiles/lwl/.env
sed -i 's|^FEISHU_APP_SECRET=.*|FEISHU_APP_SECRET=REPLACE_WITH_APP_SECRET|' ~/.hermes/profiles/lwl/.env

grep ^FEISHU_ ~/.hermes/profiles/lwl/.env
```

> 一定是新 App 的 ID/Secret。沿用 default 那个 App 会两个 gateway 抢一个连接。

## 3. 启动新 profile 的 gateway

```bash
# 跟 default gateway 一样，清空 SOCKS/HTTP 代理 env 启动
setsid env -i HOME=$HOME PATH=$PATH USER=$USER LANG=C.UTF-8 \
  bash -c "lwl gateway run </dev/null > /tmp/hermes-lwl-gateway.log 2>&1" \
  </dev/null >/dev/null 2>&1 &

sleep 10
lwl gateway status
tail -10 /tmp/hermes-lwl-gateway.log
```

成功标志：

```
[Lark] [INFO] connected to wss://msg-frontier.feishu.cn/ws/v2?...
```

`hermes profile list` 应该看到两个都是 `running`：

```
 Profile     Model                        Gateway      Alias
 ◆default    custom/claude-opus-4.7-1m-   running      —
  lwl        custom/claude-opus-4.7-1m-   running      lwl
```

## 4. 后续维护

```bash
# 进 lwl profile 的 CLI（独立会话/记忆/skills）
lwl -z "ping"
lwl

# 切换 default profile
hermes profile use default
hermes profile use lwl

# 查/编辑 lwl 的配置
lwl config show
lwl config set model.default custom/claude-opus-4.7
$EDITOR ~/.hermes/profiles/lwl/.env

# 删除 profile（注意会删除其会话/记忆/skills）
hermes profile delete lwl
```

## 5. 资源/隔离说明

| 维度 | 隔离？ |
|---|---|
| 飞书 App | ✅ 各自一份 App ID/Secret |
| `.env` / `config.yaml` | ✅ `~/.hermes/profiles/<name>/` |
| 会话 / 记忆 / skills | ✅ 各自一份 |
| Gateway 进程 | ✅ 完全独立 |
| 模型 endpoint / API key | ❌ 默认共享（用 clone 复制了一份，可以各改各的） |
| 上游 LLM 配额 | ❌ 共享（同一个 endpoint） |

适合：一台机器跑几个机器人（公司号 / 个人号 / 测试号），各自有独立人格、记忆、群隔离。

## 6. 真实例子

`harness` 这台机器现在跑了两个 Hermes 飞书机器人：

| Profile | Wrapper | Feishu App 显示名 | App ID 前缀 |
|---|---|---|---|
| `default` | `hermes` | Hermes | `cli_aa83466156…` |
| `lwl` | `lwl` | 李万里的智能助手 | `cli_aa80d37d20…` |

两个 gateway 进程互相不知道对方存在，共享 `~/.hermes/skills/`（如果想各跑各的 skill，新 profile 用 `--no-skills` 创建）。
