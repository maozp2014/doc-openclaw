# OpenClaw 新人上手指南

> 本文档为 OpenClaw 项目的新人上手指南，基于源码分析自动生成。

---

## 📌 项目概述

**OpenClaw** 是一个运行在你自己设备上的个人 AI 助手，通过你日常使用的消息渠道（Telegram、WhatsApp、Discord、飞书、微信等）与你交互。核心是一个 **Gateway（网关）**，负责连接 AI 模型和各种消息渠道，同时提供文件管理、代码执行、网页浏览等工具能力。

**官网**：https://openclaw.ai
**文档**：https://docs.openclaw.ai
**Discord**：https://discord.gg/clawd
**仓库**：https://github.com/openclaw/openclaw

---

## 🗂️ 项目结构总览

```
openclaw/
├── src/                    # 核心源码（TypeScript）
│   ├── agents/             # Agent 运行引擎（LLM 调用、Session 管理、Tool 执行）
│   ├── channels/           # 消息渠道适配层（Telegram/Discord/Slack/Signal/WhatsApp...）
│   ├── channels/plugins/   # 渠道插件核心框架（注册、分发、消息动作）
│   ├── gateway/            # WebSocket 网关（接收客户端请求、路由 Agent 事件）
│   ├── routing/            # 会话路由（Session Key 解析、Agent 绑定规则）
│   ├── auto-reply/         # 自动回复逻辑（心跳、HEARTBEAT、Reply Tags）
│   ├── cron/               # 定时任务调度器
│   ├── config/             # 配置加载与校验
│   ├── skills/             # 内置 Skill 系统
│   ├── plugins/            # 插件系统
│   ├── infra/              # 基础设施（环境变量、日志、加密、Backoff 等）
│   └── ...
├── extensions/             # 扩展渠道插件（飞书、微信、企业微信、QQ 等）
├── apps/                   # 桌面/移动端应用（macOS/iOS/Android）
├── ui/                     # Web 控制面板
├── docs/                   # 项目文档（Markdown）
├── skills/                 # 内置 Skills
├── scripts/                # 构建与运维脚本
└── test/                   # 测试套件
```

### 核心目录详解

| 目录 | 职责 |
|-------|------|
| `src/agents/` | Agent 的核心引擎：Session 管理、LLM 调用、Tool 注册与执行、上下文压缩 |
| `src/channels/` | 所有消息渠道的适配器（Telegram/Discord/WhatsApp/Web...） |
| `src/channels/plugins/` | 渠道插件的统一抽象：注册、分发、消息动作（send/reaction/pin...） |
| `src/gateway/` | WebSocket 网关主程序：处理客户端请求（`req:agent` / `req:chat`）并推送 Agent 事件流 |
| `src/routing/` | Session Key 路由：根据 channel + accountId + peer 解析走哪个 Agent |
| `src/cron/` | 定时任务：基于 cron 表达式调度，支持 subagent 隔离执行 |
| `src/skills/` | Skill 加载与执行框架 |
| `extensions/` | 非核心渠道插件：飞书(feishu)、微信(openclaw-weixin)、QQ(qqbot) 等 |
| `ui/` | Web 控制面板（WebSocket 客户端） |

---

## ⚙️ 本地开发环境搭建

### 环境要求

- **Node.js ≥ 22**（推荐使用 Node 22+）
- **pnpm**（推荐）或 npm / bun
- macOS / Linux / Windows (WSL2)

### 快速开始

```bash
# 克隆仓库
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# 安装依赖
pnpm install

# 类型检查 + 构建
pnpm build

# 类型检查（不构建）
pnpm tsgo

# 代码检查（lint + format）
pnpm check

# 运行测试
pnpm test

# 格式化修复
pnpm format:fix

# CLI 开发模式运行（热重载）
pnpm dev
# 或
pnpm openclaw <command>
```

### 环境变量

关键环境变量（参考 `~/.profile` 或 `.env`）：

```bash
# 模型 API Keys
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
DASHSCOPE_API_KEY=sk-...

# Gateway 配置
OPENCLAW_GATEWAY_TOKEN=your-token    # WebSocket 认证 Token
OPENCLAW_WORKSPACE=~/.openclaw/workspace  # 工作区目录

# 日志
DEBUG=openclaw:*
```

### 测试

```bash
# 运行所有测试
pnpm test

# 带覆盖率
pnpm test:coverage

# 低内存机器（限制并发）
OPENCLAW_TEST_PROFILE=low OPENCLAW_TEST_SERIAL_GATEWAY=1 pnpm test

# 真实 API 测试（需要真实 Keys）
CLAWDBOT_LIVE_TEST=1 pnpm test:live
```

---

## 🧠 核心概念

### 1. Gateway（网关）

OpenClaw 的核心进程，是一个 **long-lived WebSocket 服务**，负责：
- 维持所有消息渠道的长连接（如 WhatsApp Web、Discord Bot）
- 接收客户端（CLI / macOS App / Web UI）的 `req:agent` 请求
- 推送 Agent 生成的流式事件（thinking / text / tool_use 等）
- 管理所有 Session 状态和定时任务

启动方式：
```bash
openclaw gateway --port 18789 --verbose
```

### 2. Session（会话）

OpenClaw 以 **Session** 为单位管理对话上下文：

- **Session Key 格式**：`agent:<agentId>:<channel>:<peerKind>:<peerId>`
  - 例如：`agent:main:feishu:direct:ou_xxx`
- Session 数据存储在：`~/.openclaw/agents/<agentId>/sessions/`
- 对话历史存储在：`~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`

### 3. Agent（智能体）

一个 Agent 对应一个 AI 角色，包含：
- **System Prompt**（身份、行为规范）
- **Workspace**（工作区文件：SOUL.md、AGENTS.md、MEMORY.md 等）
- **Skills**（技能定义）
- **Model 配置**（使用哪个 LLM）
- **Tools**（可用工具）

### 4. Channel（渠道）

Channel 是消息的来源/去向，支持 20+ 渠道：

| 渠道 | 代码位置 |
|------|---------|
| Telegram | `src/telegram/` |
| Discord | `src/discord/` |
| Slack | `src/slack/` |
| Signal | `src/signal/` |
| WhatsApp | `src/web/` |
| 飞书 | `extensions/feishu/` |
| 微信 | `extensions/openclaw-weixin/` |
| QQ | `extensions/qqbot/` |
| 企业微信 | `extensions/wecom/` |

每个渠道通过 **Channel Plugin** 接入，共享 `src/channels/plugins/` 框架。

### 5. Skills（技能）

Skill 是可被 Agent 调用的工具能力包，定义在 `SKILL.md` 文件中：

```markdown
# Skill 名称
描述：何时触发这个 skill
指令：
1. 执行步骤...
2. 使用 Tool ...
```

Agent 在 system prompt 中包含 workspace 下所有 skills 的内容。

### 6. Cron（定时任务）

OpenClaw 支持定时任务（基于 cron 表达式），可指定：
- 独立 Session 执行（`sessionTarget: isolated`）
- 独立 Subagent 执行
- 定时发送到指定渠道

---

## 🔑 核心模块详解

### 消息处理流程（飞书 → Agent）

```
飞书服务器
  ↓ webhook / websocket
飞书 Channel Adapter（extensions/feishu/）
  ↓ dispatchInboundMessage()
路由层（src/routing/resolve-route.ts）
  ↓ sessionKey = agent:main:feishu:direct:ou_xxx
会话层（src/sessions/）
  ↓ 加载历史 + Bootstrap（SOUL.md/AGENTS.md）
Agent 启动（src/agents/pi-embedded-runner/run.ts → runEmbeddedPiAgent）
  ↓ 系统提示词 = 内置 + Workspace Files + Skills + Tools
LLM 推理（src/agents/pi-embedded-subscribe.ts → subscribeEmbeddedPiSession）
  ├── tool_use 拦截 → Tool Handler → Tool Result 回注
  └── text stream → 累积 → 切块
响应发送（src/channels/plugins/message-actions.ts）
  ↓ channel: "feishu", action: "send"
飞书 Open API → im/v1/messages/reply
```

### Session 路由优先级

1. `binding.peer` — 精确匹配 channel + account + peer
2. `binding.peer.parent` — 线程场景
3. `binding.guild+roles` — Discord 多角色
4. `binding.account` — 整个账号
5. `binding.channel` — 整个渠道
6. `default` — 默认 main agent

### Tool 调用机制

```
LLM 返回 tool_use 块
  ↓
pi-embedded-subscribe 拦截
  ↓
Tool Policy 检查（sandbox/path/exec policy）
  ↓
Tool Handler 执行（exec / read / message / browser 等）
  ↓
结果序列化 → 注回 LLM messages[]
  ↓
LLM 继续生成
```

---

## 🛠️ 常用开发命令

```bash
# 启动 Gateway
openclaw gateway run

# 查看 Gateway 状态
openclaw status

# 列出所有渠道状态
openclaw channels status

# 诊断问题
openclaw doctor

# 安装 Skill
openclaw skills install <skill-name>

# 列出定时任务
openclaw cron list

# 手动触发定时任务
openclaw cron run <job-id>

# 发送消息
openclaw message send --to <receiver> --message "Hello"

# 与 Agent 对话
openclaw agent --message "Your question"

# 交互式聊天
openclaw chat

# 配置
openclaw config get <key>
openclaw config set <key> <value>

# 更新
npm install -g openclaw@latest
```

---

## 📂 关键配置文件

| 文件路径 | 作用 |
|---------|------|
| `~/.openclaw/openclaw.json` | 主配置文件（Gateway、Agents、Channels、Cron 等） |
| `~/.openclaw/agents/<agentId>/sessions/sessions.json` | Session 索引 |
| `~/.openclaw/agents/<agentId>/sessions/<id>.jsonl` | 完整对话历史 |
| `~/.openclaw/workspace/SOUL.md` | Agent 灵魂/性格定义 |
| `~/.openclaw/workspace/AGENTS.md` | 工作区规范 |
| `~/.openclaw/workspace/MEMORY.md` | Agent 长期记忆 |
| `~/.openclaw/workspace/USER.md` | 用户信息 |
| `~/.openclaw/workspace/HEARTBEAT.md` | 心跳检查任务 |
| `~/.openclaw/cron/jobs.json` | 定时任务配置 |

---

## 🏗️ 如何添加新功能

### 添加新的消息渠道

1. 在 `src/channels/` 或 `extensions/` 下创建新的 channel adapter
2. 实现 `ChannelPlugin` 接口（`connect / disconnect / send / receive`）
3. 在 `src/channels/plugins/catalog.ts` 注册
4. 添加配置到 `openclaw.json` 的 `channels` 字段
5. 编写测试：`src/channels/<name>/*.test.ts`

### 添加新的 Tool

1. 在 `src/agents/tools/` 或 `skills/` 下创建 Tool 定义
2. 实现 Tool Handler（参数解析、执行逻辑、结果格式化）
3. 在 Tool Catalog 注册
4. 添加 Tool Policy 规则

### 添加新的 Skill

1. 在 `~/.openclaw/workspace/skills/<skill-name>/` 创建 `SKILL.md`
2. 定义 `description`（触发场景）和 `argument-hint`
3. 编写 Agent 指令（分步骤）
4. Agent 在 system prompt 中自动加载

---

## 📖 文档速查

| 文档 | 内容 |
|------|------|
| `docs/concepts/architecture.md` | 整体架构图 |
| `docs/concepts/session.md` | Session 管理规则 |
| `docs/concepts/agent.md` | Agent 系统详解 |
| `docs/gateway/index.md` | Gateway 协议与 API |
| `docs/channels/index.md` | 各渠道配置指南 |
| `docs/concepts/multi-agent.md` | 多 Agent 系统 |
| `docs/concepts/memory.md` | 记忆系统 |
| `VISION.md` | 项目愿景与方向 |
| `CONTRIBUTING.md` | 贡献指南 |
| `CLAUDE.md` | 开发者注意事项（AI 友好） |

---

## 🚀 新人常见问题

**Q：Gateway 起不来，端口被占用？**
```bash
# 查看谁在用 18789 端口
ss -ltnp | grep 18789
# 或者
lsof -i :18789
```

**Q：飞书/Telegram 连接正常但 Agent 不回复？**
```bash
openclaw doctor
# 检查 API Keys 是否配置、模型额度是否充足
```

**Q：Session 历史太长了？**
- Agent 会自动做上下文压缩（Compaction）
- 也可以手动：`openclaw sessions compact <session-id>`

**Q：如何调试？**
```bash
# 开启 verbose 日志
openclaw gateway --verbose --log-level debug

# 查看实时日志
tail -f ~/.openclaw/logs/gateway.log
```

---

*本文档由 Understand-Anything 自动生成，如有过时或错误之处请提交 PR 修正。*
