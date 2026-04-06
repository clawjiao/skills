---
name: add-agent
description: >
  添加新的 OpenClaw Agent 并绑定飞书机器人，实现多 Agent 多机器人路由。
  使用场景：(1) 创建新的 Agent 并关联飞书机器人 (2) 配置多账号路由和会话隔离 (3) 为已有 Agent 添加新的飞书机器人绑定。
  触发关键词：添加 agent、新建机器人、绑定飞书、多机器人、多 agent、add agent、new bot。
---

# 添加 Agent 并绑定飞书机器人

## 目标

创建新的飞书机器人 Agent，实现：
- 每个机器人独立 workspace、独立上下文
- 主 Agent 可调度子 Agent
- 账号 + channel + 用户粒度隔离
- **默认访问策略：`allowlist`（仅白名单用户可用）**

---

## 前置检查

### 1. 飞书插件是否安装

```bash
# 检查插件列表
openclaw config get plugins.installs | grep openclaw-lark
```

如果未安装，执行：

```bash
npx -y @larksuite/openclaw-lark install
```

### 2. 收集必要信息

向用户确认以下内容：
- **Agent ID**：如 `copywriter`、`data-crawler`（唯一标识，小写字母+连字符，用于内部路径和配置 key）
- **Agent 名称**：如「文案编辑」、「数据抓取」（展示用）
- **飞书 App ID**：格式 `cli_xxx`
- **飞书 App Secret**：用于认证
- **机器人名称**：飞书聊天列表中显示的名称

> 📌 **访问策略**：固定为 `allowlist`（仅白名单用户可用）。
> 📌 **用户 open_id**：配置时先使用占位符（如 `["pending"]`），待用户向机器人发送打招呼消息后，从日志或消息上下文中提取真实的 `ou_xxx` 并更新配置。

---

### ⚠️ 关键：open_id 与飞书机器人的对应关系

**每个飞书机器人是一个独立的飞书应用（有独立的 App ID / App Secret）。同一个用户在不同的飞书机器人（应用）下，飞书平台会分配不同的 `open_id`。**

| 飞书机器人应用 | App ID | account-id (Agent ID) | 教主在该机器人下的 open_id |
|---|---|---|---|
| 主助手 | `cli_a9478ad6f8b95bd4` | `main` | `ou_115923abeea148401bfa2af6076aacb9` |
| 数据机器人 | `cli_a944d19ffd78dcef` | `data-crawler` | `ou_132b3821de4b35fda2dd7d9d5b83f28a` |
| 文案助手(新) | `cli_xxx_NEW` | `copywriter` | `ou_xxx_NEW`（需实际对话获取） |

**核心原则：**
1. **一一对应**：`channels.feishu.accounts.<account-id>.allowFrom` 里的 `ou_xxx`，必须是用户在该**特定飞书机器人**下的 open_id。
2. **不可复用**：不能把 `main` 机器人的 `ou_115923...` 填到 `copywriter` 机器人的 `allowFrom` 里，否则教主无法与文案助手对话（因为 open_id 不匹配）。
3. **准确获取**：必须通过用户与新机器人对话后的日志来确定该机器人的专属 open_id。

**获取准确的 open_id：**
1. 用户在飞书上与**新机器人**发送一条消息（如「你好」）
2. 查看 OpenClaw 日志：
   ```bash
   tail -f /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | grep "feishu"
   ```
3. 日志格式：`feishu[<account-id>] DM | <ou_xxx> [msg:om_xxx]`
   - 例如：`feishu[copywriter] DM | ou_999abc... [msg:om_xxx]`
4. 将日志中的 `<ou_xxx>` 填入配置中的 `allowFrom`。

---

## 操作步骤

### 步骤 1：创建独立 workspace

```bash
AGENT_ID="<agent-id>"
WORKSPACE="$HOME/.openclaw/workspace-${AGENT_ID}"
mkdir -p "$WORKSPACE"

# 创建 AGENTS.md（工作区标识）
cat > "$WORKSPACE/AGENTS.md" << EOF
# AGENTS.md - ${AGENT_ID}

## 身份
- **名称：** <agent-name>
- **角色：** <描述该 agent 的职责>
- **Owner：** 教主
EOF

# 创建 SOUL.md（核心人格）
cat > "$WORKSPACE/SOUL.md" << EOF
# SOUL.md - ${AGENT_ID}

<根据 agent 职责描述其核心能力、工作原则、输出风格>
EOF

# 创建 USER.md（教主信息继承）
cp "$HOME/.openclaw/workspace/USER.md" "$WORKSPACE/USER.md"

# 创建 IDENTITY.md
cat > "$WORKSPACE/IDENTITY.md" << EOF
# IDENTITY.md

- **Name:** ${AGENT_ID}
- **Role:** <职责描述>
- **Master:** 教主
EOF
```

### 步骤 2：创建飞书机器人（二选一）

**方式 A：使用配置页面（推荐首次接入或批量创建）**

访问：`https://open.feishu.cn/page/openclaw?form=multiAgent`

- 填写机器人名称
- 页面会返回 `App ID` 和 `App Secret`
- **提醒用户发送这两个参数给你**（用于配置）

**方式 B：使用已有机器人**

如果用户已在飞书开放平台创建应用，直接获取：
- App ID（`cli_xxx`）
- App Secret

### 步骤 2b：引导用户打招呼以提取 open_id

在用户完成飞书机器人创建并提供 App ID / App Secret 后，执行以下流程：

1. **先完成步骤 3 的配置**（将 `allowFrom` 设为 `["pending"]` 占位符）。
2. 重启 Gateway：`openclaw gateway restart`。
3. **引导用户操作**：
   - 在飞书客户端中搜索机器人名称（或通过开发者小组手入口）。
   - 打开与该机器人的私聊窗口。
   - 发送一条打招呼消息（如「你好」或「hi」）。
4. **从日志中提取 open_id**：
   ```bash
   grep "sender_id\|peer.*id" /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | grep "<account-id>" | tail -1
   ```
   或直接在飞书消息上下文中查看 `[message_id=..., sender=ou_xxx]` 元数据。
5. **更新配置**：将提取到的 `ou_xxx` 填入 `allowFrom` 数组，替换 `["pending"]`。
6. **再次重启 Gateway**：`openclaw gateway restart`。

> 📌 如果用户已提前提供 `open_id`（例如从其他机器人会话中复制），可跳过此步骤，直接填入 `allowFrom`。

### 步骤 3：配置 OpenClaw

配置文件：`~/.openclaw/openclaw.json`

需要修改 **5 个部分**：

#### 3a. 会话隔离策略（全局设置）

确保以下配置存在（多账号必须）：

```json
{
  "session": {
    "dmScope": "per-account-channel-peer"
  }
}
```

同时推荐启用飞书渠道优化：

```bash
openclaw config set channels.feishu.streaming true
openclaw config set channels.feishu.threadSession true
openclaw config set channels.feishu.footer.elapsed true
openclaw config set channels.feishu.footer.status true
```

#### 3b. agents.list — 注册新 Agent

```json
{
  "agents": {
    "list": [
      {
        "id": "<agent-id>",
        "name": "<agent-name>",
        "workspace": "/Users/mac/.openclaw/workspace-<agent-id>",
        "default": false
      }
    ]
  }
}
```

> ⚠️ `workspace` 必须使用**绝对路径**

#### 3c. channels.feishu.accounts — 添加飞书账号

```json
{
  "channels": {
    "feishu": {
      "accounts": {
        "<account-id>": {
          "appId": "cli_xxx",
          "appSecret": "<app-secret-or-secret-ref>",
          "name": "<机器人名称>",
          "botName": "<机器人名称>",
          "dmPolicy": "allowlist",
          "allowFrom": ["pending"]
        }
      }
    }
  }
}
```

**参数说明：**
- `<account-id>`：通常与 `<agent-id>` 保持一致，便于管理。
- `dmPolicy`：固定为 `"allowlist"`（仅白名单用户可用）。
- `allowFrom`：初次配置时填入 `["pending"]` 作为占位符。待用户打招呼后，替换为真实的 `open_id`（格式 `ou_xxx`）。

> ⚠️ `pending` 仅用于首次配置时的占位，实际使用时必须替换为真实 `open_id`，否则机器人会拒绝所有消息（日志中会出现 `not in DM allowlist` 错误）。

**App Secret 存储方式（二选一）：**
- 明文字符串：`"appSecret": "<secret>"`（简单但不安全）
- 引用 secrets 管理器：
  ```json
  "appSecret": {
    "source": "file",
    "provider": "lark-secrets",
    "id": "/lark/accounts/<account-id>/appSecret"
  }
  ```

#### 3d. bindings — 添加路由绑定

```json
{
  "bindings": [
    {
      "agentId": "<agent-id>",
      "match": {
        "channel": "feishu",
        "accountId": "<account-id>"
      }
    }
  ]
}
```

> 📌 `accountId` 应与 `channels.feishu.accounts` 中的 key 一致。通常使用 `<agent-id>` 作为 accountId 以保持对应关系。

#### 3e. 主 Agent 的 subagents 允许列表

确保主 Agent（`main`）可以调度新 Agent：

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "default": true,
        "subagents": {
          "allowAgents": ["<agent-id>", "其他已有agent"]
        }
      }
    ]
  }
}
```

### 步骤 4：应用配置并重启

**方式 A：直接重启（推荐）**

```bash
openclaw gateway restart
```

**方式 B：使用 gateway config.patch（需当前会话已配对）**

使用 `gateway` 工具的 `config.patch` action 提交增量变更。

### 步骤 5：验证与获取 sessionKey

#### 5a. 验证机器人连接

1. 在飞书上私聊新机器人，发送一条测试消息（如「你好」）
2. 检查日志确认路由正确：
   ```bash
   tail -f /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | grep "<agent-id>"
   ```
3. 确认无 `not in DM allowlist` 错误

**如果配置时 `allowFrom` 仍为 `["pending"]`：**
1. 用户发送打招呼消息后，日志中会出现 `not in DM allowlist` 错误，但会显示发送者的 `sender_id`（即 `ou_xxx`）。
2. 提取该 `ou_xxx`，更新 `allowFrom` 数组。
3. 重启 Gateway。
4. 用户再次发送消息，确认无错误且机器人正常响应。

#### 5b. 获取 sessionKey

新 Agent 首次收到消息后，会创建会话。获取 sessionKey：

```bash
# 列出所有 agent 的会话
openclaw sessions --agent <agent-id> --json

# 或查看所有 agent
openclaw sessions --all-agents --json
```

返回示例：
```json
{
  "sessionKey": "<agent-id>:feishu:ou_xxx",
  "agentId": "<agent-id>",
  "lastActivity": "2026-04-04T08:00:00Z"
}
```

**sessionKey 格式**：`<agent-id>:<channel>:<peer-id>`

例如：`data-crawler:feishu:ou_132b3821de4b35fda2dd7d9d5b83f28a`

#### 5c. 更新主 Agent 记忆

将新 Agent 的 sessionKey 和调度信息写入主 Agent 的记忆文件 `~/.openclaw/workspace/AGENTS-REGISTRY.md`：

```markdown
## Agent 注册表

| Agent ID | 名称 | sessionKey | 职责 |
|----------|------|------------|------|
| main | 主助手 | main:feishu:ou_xxx | 调度中枢 |
| data-crawler | 数据抓取 | data-crawler:feishu:ou_xxx | 爬取数据 |
| <agent-id> | <名称> | <sessionKey> | <职责> |
```

同时更新 `~/.openclaw/workspace/MEMORY.md` 中的「智能体调度」章节。

---

## 主 Agent 调用子 Agent

主 Agent 可通过以下方式调用子 Agent：

### 方式 1：sessions_send（直接发送消息）

```bash
openclaw sessions_send \
  --sessionKey "<agent-id>:feishu:ou_xxx" \
  --message "<指令内容>"
```

或在代码/对话中调用 `sessions_send` 工具。

### 方式 2：sessions_spawn（创建临时任务）

```bash
openclaw sessions_spawn \
  --runtime subagent \
  --agentId <agent-id> \
  --task "<任务描述>"
```

### 方式 3：subagents 调度

```bash
# 列出当前会话的子 agent
openclaw subagents list

# 向指定子 agent 发送指令
openclaw sessions_send --sessionKey "<key>" --message "<msg>"
```

---

## 完整配置示例

```json
{
  "session": {
    "dmScope": "per-account-channel-peer"
  },
  "agents": {
    "list": [
      {
        "id": "main",
        "name": "主助手",
        "default": true,
        "workspace": "/Users/mac/.openclaw/workspace",
        "subagents": {
          "allowAgents": ["data-crawler", "project-manager", "code-engineer"]
        }
      },
      {
        "id": "data-crawler",
        "name": "数据抓取",
        "default": false,
        "workspace": "/Users/mac/.openclaw/workspace-data-crawler"
      }
    ]
  },
  "channels": {
    "feishu": {
      "accounts": {
        "main": {
          "appId": "cli_a9478ad6f8b95bd4",
          "appSecret": { "source": "file", "provider": "lark-secrets", "id": "/lark/appSecret" },
          "name": "大龙虾助手",
          "botName": "大龙虾助手",
          "dmPolicy": "allowlist",
          "allowFrom": ["ou_115923abeea148401bfa2af6076aac91"]
        },
        "data-crawler": {
          "appId": "cli_a944d19ffd78dcef",
          "appSecret": "<secret>",
          "name": "数据抓取机器人",
          "botName": "数据抓取机器人",
          "dmPolicy": "allowlist",
          "allowFrom": ["ou_132b3821de4b35fda2dd7d9d5b83f22a"]
        }
      },
      "streaming": true,
      "threadSession": true
    }
  },
  "bindings": [
    {
      "agentId": "main",
      "match": { "channel": "feishu", "accountId": "main" }
    },
    {
      "agentId": "data-crawler",
      "match": { "channel": "feishu", "accountId": "data-crawler" }
    }
  ],
  "tools": {
    "profile": "full",
    "sessions": { "visibility": "all" },
    "agentToAgent": { "enabled": true }
  }
}
```

---

## 常见问题排查

| 问题 | 可能原因 | 排查步骤 |
|------|----------|----------|
| 机器人无响应 | dmPolicy 限制 / 未绑定 | 检查 `dmPolicy` 和 `allowFrom`，验证 `bindings` |
| `not in DM allowlist` | `allowFrom` 未包含用户的 `open_id` 或仍为占位符 | 按「步骤 2b」提取真实 `open_id` 并更新配置 |
| WebSocket 未连接 | 事件订阅未配置 | 飞书后台 → 应用 → 事件与回调 → 确保启用「长连接模式」 |
| 消息路由到错误 agent | binding accountId 不匹配 | 检查 `bindings[].match.accountId` 是否与账号 key 一致 |
| 子 Agent 无记忆 | 未设置 dmScope | 确认 `session.dmScope = "per-account-channel-peer"` |
| sessions_send 失败 | sessionKey 错误 / agent 未运行 | 验证 sessionKey 格式，检查 `openclaw sessions --agent <id>` |
| 403 Forbidden | App Secret 错误 / 权限不足 | 重新核对 App ID/Secret，检查飞书应用权限 |

**快速诊断命令：**

```bash
# 检查网关状态
openclaw gateway status

# 检查特定 agent 配置
openclaw config get agents.list | grep "<agent-id>" -A 5

# 检查飞书账号配置
openclaw config get channels.feishu.accounts

# 查看实时日志
tail -f /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | grep -E "feishu|<agent-id>"
```

---

## 安全注意事项

1. **禁止在 skill 文件或公开文档中硬编码 App Secret**，使用 `{app_secret}` 占位符
2. **优先使用 allowlist 白名单**，避免 `open` 策略
3. **定期轮换 App Secret**（飞书开放平台可重置）
4. **隔离 workspace**：每个 Agent 有独立目录，避免交叉污染
5. **会话隔离**：`dmScope = "per-account-channel-peer"` 确保不同机器人之间记忆不混淆
