# 如何为 OpenClaw 配置网络代理

## 概述

OpenClaw 是一个运行在本地服务器上的个人 AI 助手框架，通过 Gateway 进程对外提供服务，支持 Telegram、WhatsApp、Discord、Slack 等多种聊天频道。它依赖网络访问 AI 模型提供商的 API（如 OpenAI、Anthropic、MiniMax 等）以及各聊天平台的 API。

在中国大陆或网络受限的环境中，以下服务可能无法直接访问：

- OpenAI、Anthropic 等 AI 模型 API
- Telegram Bot API（`api.telegram.org`）
- 其他境外 AI 服务商接口

因此需要为 OpenClaw 配置网络代理，使其能够正常连接上述服务。

**本文档基于以下实际环境编写：**

- 操作系统：Ubuntu Linux
- OpenClaw 版本：2026.3.24（安装包版本，`openclaw --version` 报告）；服务运行版本：2026.4.5（由 `OPENCLAW_SERVICE_VERSION` 环境变量标识，可能因热更新而高于安装包版本）
- 安装路径：`/home/chris/.npm-global/lib/node_modules/openclaw`
- 配置目录：`~/.openclaw/`
- 运行方式：systemd 用户服务（`openclaw-gateway.service`）

---

## 环境要求

- Node.js 22.14+ 或 24（推荐）
- OpenClaw 已通过 `npm install -g openclaw` 安装并完成 `openclaw onboard`
- 本地已有可用的 HTTP 或 SOCKS5 代理服务（如 Clash、V2Ray、Shadowsocks 等）
- systemd 用户服务（`openclaw-gateway.service`）已安装

验证安装情况：

```bash
# 检查 openclaw 版本
openclaw --version

# 检查 gateway 服务状态
openclaw gateway status

# 查看服务文件位置
systemctl --user status openclaw-gateway
```

---

## 代理配置方法

OpenClaw 的代理配置分为三个层次，根据需求选择一种或组合使用：

1. **方法一：通过 systemd drop-in 文件设置全局代理环境变量**（推荐，覆盖所有出站请求）
2. **方法二：通过 openclaw.json 为特定 AI 模型 Provider 配置代理**（精细控制）
3. **方法三：为特定聊天频道（如 Telegram）配置代理**（仅针对该频道）

---

### 方法一：systemd drop-in 文件（推荐）

这是最简洁、覆盖最广的方式。通过 systemd drop-in 配置文件向 Gateway 服务注入代理环境变量，所有 Node.js 进程发出的 HTTP/HTTPS 请求都会走代理。

#### 步骤 1：创建 drop-in 目录

```bash
mkdir -p ~/.config/systemd/user/openclaw-gateway.service.d
```

#### 步骤 2：创建代理配置文件

```bash
cat > ~/.config/systemd/user/openclaw-gateway.service.d/proxy.conf << 'EOF'
[Service]
Environment=HTTPS_PROXY=http://127.0.0.1:1081
Environment=HTTP_PROXY=http://127.0.0.1:1081
Environment=NO_PROXY=localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16
EOF
```

将 `http://127.0.0.1:1081` 替换为你实际的代理地址和端口。若代理为 SOCKS5 协议，格式为 `socks5://127.0.0.1:1080`。

**常见代理地址示例：**

| 代理软件 | 默认地址 | 典型端口 |
|----------|----------|----------|
| Clash（HTTP 端口） | `http://127.0.0.1:7890` | 7890 |
| Clash（SOCKS5 端口） | `socks5://127.0.0.1:7891` | 7891 |
| V2Ray（HTTP 端口） | `http://127.0.0.1:1081` | 1081 |
| Shadowsocks | `socks5://127.0.0.1:1080` | 1080 |

#### 步骤 3：重新加载 systemd 并重启服务

```bash
# 重新加载 systemd 配置
systemctl --user daemon-reload

# 重启 OpenClaw Gateway
systemctl --user restart openclaw-gateway

# 确认服务状态
systemctl --user status openclaw-gateway
```

#### 注意事项

- 若系统仅有 HTTPS 出站需求（如 OpenAI、Anthropic、Telegram 等 API 均为 HTTPS），只设置 `HTTPS_PROXY` 即可，无需设置 `HTTP_PROXY`。若同时有 HTTP 出站需求，才需要一并添加 `HTTP_PROXY`。
- `NO_PROXY` 变量用于排除不需要走代理的地址，本地内网地址和内部服务地址应当加入其中。若未设置 `NO_PROXY` 且已设置 `HTTP_PROXY`，访问内网 HTTP 地址（如 `http://10.x.x.x`）时请求将尝试经代理发出，可能导致连接失败。
- 若使用的是本机局域网内的 AI 服务（如 `baseUrl: "http://10.0.3.248:3000/api/v1"`），请确保将其加入 `NO_PROXY`，避免走代理失败。
- 修改 drop-in 文件后，务须执行 `daemon-reload` 和 `restart`，否则配置不生效。

---

### 方法二：在 openclaw.json 中为 AI 模型 Provider 配置代理

> **适用版本：openclaw 2026.3.24（以下配置字段可能因版本而异，请以实际版本的文档为准）**

当你只希望特定 AI 模型 Provider 走代理时，可以在 `~/.openclaw/openclaw.json` 中的 `models.providers` 配置块里设置 `request.proxy`。

> 注意：以下 JSON 示例均为示意片段，`"models": [...]` 中的 `[...]` 表示保留原有 models 数组内容，实际编辑时请保留原有配置，仅添加 `request.proxy` 字段。建议使用 `openclaw config set` 命令或 `openclaw configure` 向导代替手动编辑 JSON，以避免格式错误。

OpenClaw 支持两种模式：

- `env-proxy`：使用进程环境变量中的 `HTTP_PROXY` / `HTTPS_PROXY`（与方法一配合使用时自动生效）
- `explicit-proxy`：直接指定代理 URL

#### 方式 A：使用环境变量代理（env-proxy 模式）

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "openai": {
        "baseUrl": "https://api.openai.com/v1",
        "apiKey": "sk-...",
        "api": "openai-completions",
        "request": {
          "proxy": {
            "mode": "env-proxy"
          }
        },
        "models": [...]
      }
    }
  }
}
```

#### 方式 B：直接指定代理地址（explicit-proxy 模式）

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "openai": {
        "baseUrl": "https://api.openai.com/v1",
        "apiKey": "sk-...",
        "api": "openai-completions",
        "request": {
          "proxy": {
            "mode": "explicit-proxy",
            "url": "http://127.0.0.1:1081"
          }
        },
        "models": [...]
      }
    }
  }
}
```

#### 使用 CLI 查看/修改配置

OpenClaw 提供 CLI 工具操作配置文件，避免手动编辑 JSON 产生格式错误：

```bash
# 查看当前配置文件路径
openclaw config file

# 查看某个配置值
openclaw config get models.providers.openai.request.proxy

# 通过交互式向导配置
openclaw configure
```

配置修改后，重启 Gateway 使其生效：

```bash
systemctl --user restart openclaw-gateway
```

---

### 方法三：为 Telegram 频道配置代理

> **适用版本：openclaw 2026.3.24（以下配置字段可能因版本而异，请以实际版本的文档为准）**

若仅 Telegram Bot API 访问受阻（如 `api.telegram.org` 无法连通），可在 `openclaw.json` 的 `channels.telegram` 块中单独配置代理，而不影响其他请求。

支持 HTTP 和 SOCKS5 代理协议：

```json
{
  "channels": {
    "telegram": {
      "proxy": "socks5://127.0.0.1:1080"
    }
  }
}
```

或使用 HTTP 代理：

```json
{
  "channels": {
    "telegram": {
      "proxy": "http://127.0.0.1:1081"
    }
  }
}
```

多账号时，也可以为某个具体账号单独设置代理：

```json
{
  "channels": {
    "telegram": {
      "accounts": {
        "telegram_leon": {
          "botToken": "...",
          "proxy": "socks5://127.0.0.1:1080"
        }
      }
    }
  }
}
```

配置修改后重启服务：

```bash
systemctl --user restart openclaw-gateway
```

---

### 补充：为 npm 安装配置代理（仅影响 openclaw 更新和插件安装）

若需要在受限网络下安装或更新 openclaw 及其插件，还需要为 npm 设置代理。这与运行时代理是独立的配置：

```bash
# 设置 npm 代理（全局）
npm config set proxy http://127.0.0.1:1081
npm config set https-proxy http://127.0.0.1:1081

# 安装完成后，若不再需要可以清除
npm config delete proxy
npm config delete https-proxy
```

npm 代理配置存储在 `~/.npmrc` 文件中，可直接查看：

```bash
cat ~/.npmrc
```

---

## 验证代理是否生效

### 方法 1：查看 systemd 服务的环境变量

```bash
# 推荐：查看特定服务（含 drop-in 注入）实际加载的环境变量
systemctl --user show openclaw-gateway | grep -i proxy

# 或者查看服务进程的环境变量（替换 <PID> 为实际 PID）
ps aux | grep openclaw-gateway
cat /proc/<PID>/environ | tr '\0' '\n' | grep -i proxy
```

> 注意：`systemctl --user show-environment` 显示的是用户 systemd manager 的**全局**环境变量，并不包含 drop-in 文件通过 `Environment=` 指令注入到特定服务的变量。要验证 drop-in 代理配置是否对 `openclaw-gateway` 服务生效，应使用 `systemctl --user show openclaw-gateway | grep -i proxy`。

### 方法 2：通过 openclaw doctor 检查频道连通性

```bash
openclaw doctor
```

该命令会检查各聊天频道的连接状态。若 Telegram 状态从 `failed` 变为 `ok`，说明代理已生效。

### 方法 3：查看 Gateway 日志

```bash
# 实时跟踪日志（加 --follow 持续输出）
openclaw logs --follow

# 仅查看当前日志快照（默认行为，输出后退出）
openclaw logs

# 或直接查看日志文件（按日期命名）
tail -f /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log
```

> 注意：`openclaw logs` 默认输出当前日志快照后即退出，需加 `--follow` 才能持续实时跟踪新产生的日志。

正常连接时，日志中应无大量 `fetch failed`、`Network request failed` 等错误；代理生效时，可能会看到类似 `Using proxy http://127.0.0.1:1081` 的日志条目。

### 方法 4：向 Agent 发送测试消息

通过 Telegram 或其他频道向 Agent 发送消息，如果能正常收到回复，说明 AI 模型 API 和聊天频道代理均已生效。

也可以通过 CLI 直接测试：

```bash
# 通过 E.164 格式手机号指定会话
openclaw agent --message "你好，请回复一句话" --to +15555550123

# 或直接指定 Agent ID（无需 Telegram 账号）
openclaw agent --message "你好，请回复一句话" --agent leon
```

> 注意：`--to` 参数接受 E.164 格式的手机号（如 `+15555550123`），用于派生会话 key，与 Telegram 用户名无关。若只需测试 AI 模型连通性，推荐使用 `--agent <id>` 指定 Agent ID。

---

## 常见问题排查

### 问题 1：配置了代理后 Gateway 仍无法连接

**排查步骤：**

1. 确认代理服务本身正常运行：
   ```bash
   curl -v --proxy http://127.0.0.1:1081 https://api.telegram.org
   ```

2. 确认 drop-in 配置已被加载：
   ```bash
   systemctl --user cat openclaw-gateway
   # 应显示 proxy.conf 的内容被合并进来
   ```

3. 确认执行了 `daemon-reload` 并重启了服务：
   ```bash
   systemctl --user daemon-reload
   systemctl --user restart openclaw-gateway
   systemctl --user status openclaw-gateway
   ```

### 问题 2：内网 AI 服务（如本地部署的模型）走了代理导致连接失败

**原因：** `HTTPS_PROXY` 对所有请求生效，包括内网地址。

**解决方案：** 在 drop-in 文件中添加 `NO_PROXY` 排除内网地址：

```bash
cat > ~/.config/systemd/user/openclaw-gateway.service.d/proxy.conf << 'EOF'
[Service]
Environment=HTTPS_PROXY=http://127.0.0.1:1081
Environment=HTTP_PROXY=http://127.0.0.1:1081
Environment=NO_PROXY=localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16,172.16.0.0/12
EOF
```

然后重新加载并重启：

```bash
systemctl --user daemon-reload && systemctl --user restart openclaw-gateway
```

### 问题 3：Telegram 显示 `fetch failed`，但其他 AI 请求正常

**可能原因：** Telegram Bot API 连接问题，`api.telegram.org` 可能优先解析为 IPv6 地址，而 IPv6 出口不通。

**解决方案一：** 在 `openclaw.json` 中强制使用 IPv4：

> **适用版本：openclaw 2026.3.24（以下配置字段可能因版本而异，请以实际版本的文档为准）**

```json
{
  "channels": {
    "telegram": {
      "network": {
        "autoSelectFamily": false,
        "dnsResultOrder": "ipv4first"
      }
    }
  }
}
```

**解决方案二：** 为 Telegram 单独配置 SOCKS5 代理（参见方法三）。

**解决方案三：** 通过环境变量临时测试：

```bash
OPENCLAW_TELEGRAM_DNS_RESULT_ORDER=ipv4first openclaw gateway
```

> 注意：`OPENCLAW_TELEGRAM_DNS_RESULT_ORDER` 为 OpenClaw 内部实现所使用的环境变量，官方文档中未明确列出，可能随版本变化而失效。推荐优先使用解决方案一（修改 `openclaw.json` 中的 `channels.telegram.network.dnsResultOrder` 字段），该方式更稳定可靠。

### 问题 4：`openclaw update` 或安装插件时下载超时

npm 的运行时代理与 OpenClaw Gateway 代理是独立的。需要为 npm 单独配置代理：

```bash
npm config set proxy http://127.0.0.1:1081
npm config set https-proxy http://127.0.0.1:1081

# 更新 openclaw
npm install -g openclaw@latest

# 安装插件
openclaw plugins install @openclaw/插件名

# 完成后清理 npm 代理（可选）
npm config delete proxy
npm config delete https-proxy
```

### 问题 5：修改 openclaw.json 后配置不生效

`openclaw.json` 的修改需要重启 Gateway 才能生效：

```bash
systemctl --user restart openclaw-gateway
```

也可以验证当前配置文件是否语法正确：

```bash
openclaw config validate
```

---

## 快速参考

### 查看当前系统代理状态

```bash
# 查看环境变量
env | grep -i proxy

# 查看 openclaw 服务环境变量
systemctl --user show openclaw-gateway | grep Environment
```

### 查看 openclaw 配置文件

```bash
# 查看配置文件路径
openclaw config file

# 查看 Gateway 运行状态
openclaw gateway status

# 健康检查
openclaw health
```

### systemd drop-in 文件位置

```
~/.config/systemd/user/openclaw-gateway.service.d/proxy.conf
```

### openclaw 主配置文件位置

```
~/.openclaw/openclaw.json
```

### 日志文件位置

```
/tmp/openclaw/openclaw-YYYY-MM-DD.log
```

---

## 参考资料

- OpenClaw 官方文档：https://docs.openclaw.ai
- OpenClaw 配置参考：https://docs.openclaw.ai/gateway/configuration-reference
- OpenClaw 环境变量说明：https://docs.openclaw.ai/help/environment
- OpenClaw GitHub：https://github.com/openclaw/openclaw

