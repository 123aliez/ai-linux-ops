# AI Linux Ops — AI 驱动的 Linux 服务运维管理框架

一套面向 AI 编码助手（Claude Code / Codex CLI / Gemini CLI 等）的 **Linux 服务运维管理框架**。让 AI 助手成为你的运维搭档——它知道你的服务器跑了哪些服务、配置文件在哪、怎么重启，并且严格遵守操作边界，不会越权。

## 这是什么？

如果你有一台 Linux 服务器，上面跑着十几个对外服务（API 反代、监控面板、数据库……），每次让 AI 帮你排查问题都要从零解释环境，这个框架解决了这个问题。

**一句话**：让 AI 助手在读到你的第一句话之前，就已经完整掌握了你的服务器和服务全貌。

## 核心理念

- **上下文预加载**：AI 启动时自动读取框架文件，无需人工解释环境
- **操作边界红线**：AI 严格遵守"先确认再操作"，高危命令必须审阅
- **工具无关**：纯 Markdown 文件，Claude Code / Codex / Gemini / Cursor 通用
- **跨会话持久**：所有状态写入文件，不依赖 AI 的记忆系统
- **Codex 只读协作**：主代理负责规划、调度和执行；排障信息收集交给只读的 Codex，变更方案和变更结果由新开的 Codex 会话独立审查
- **只有框架入库**：真实服务器信息和服务文档只保存在本机（`.gitignore` 排除），仓库保持为通用模板

## 目录结构

```
ai-linux-ops/
├── CLAUDE.md              # Claude Code 入口（@ 导入规则文件）
├── AGENTS.md              # Codex 入口（含只读助手模式）
├── RULES_GLOBAL.md        # 行为规则（7 条）
├── DESC_DIR.md            # 目录管理规范（7 条）
├── .claude/settings.json  # 权限：高危命令强制确认
├── .gitignore             # 排除下面的本机状态文件
├── _template/
│   ├── SERVICE.md         # 服务文档模板
│   ├── PROGRESS.md        # 进度追踪模板
│   ├── DESC_ENV.md        # 服务器环境模板
│   └── SERVICES_INDEX.md  # 服务索引模板
│
│  以下为本机状态，不入库：
├── DESC_ENV.md            # 服务器环境描述（IP、系统、软件栈）
├── SERVICES_INDEX.md      # 全局服务索引（所有服务的地图）
├── services/              # 服务文档目录
└── *.md                   # 知识文档
```

## 文件说明

### 入口文件：`CLAUDE.md`

AI 助手启动时自动读取的入口。`CLAUDE.md` 用 `@` 导入四个核心文件；`AGENTS.md` 指示 Codex 按顺序读取它们，被 Claude 以「【任务书】」调用时则进入只读助手模式。不同 AI 工具对应不同的入口文件名：

| AI 工具 | 入口文件 |
|---------|---------|
| Claude Code | `CLAUDE.md` |
| Codex CLI | `AGENTS.md` |
| Gemini CLI | `GEMINI.md` |

Gemini CLI 可以复制 `AGENTS.md` 并重命名为 `GEMINI.md`。

### 行为规则：`RULES_GLOBAL.md`

定义 AI 助手的 7 条行为准则：

| # | 规则 | 说明 |
|---|------|------|
| 1 | 语言设定 | 按用户偏好语言回答 |
| 2 | 行动前置确认 | 涉及文件/服务器操作必须先确认 |
| 3 | 上下文解析优先 | 优先读取用户提供的文档，不用预训练数据 |
| 4 | 状态确认握手 | 每次新会话用一句话确认已加载框架 |
| 5 | 状态存储唯一性 | 只写入本框架的文件，不写工具专属配置 |
| 6 | 运维偏好 | Codex 只读信息收集与独立审查、高危命令确认（ask 规则强制）、变更后同步 |
| 7 | 网络代理降级 | 网络失败时自动通过代理重试 |

### 服务器环境：`DESC_ENV.md`

填写你的真实服务器信息：

- 服务器清单（IP、系统、CPU/内存、用途）
- SSH 访问方式
- 通用软件栈版本
- 网络架构（反代、SSL、端口规划）
- 工作目录约定

### 目录规范：`DESC_DIR.md`

7 条目录管理规则，包括：

- 全局索引优先读取
- 服务寻址锁定（确认后才操作特定服务）
- 新服务创建流程（确认 → 模板 → 填写 → 写入）
- 变更后主动询问更新文档
- 操作边界红线
- 模板只读保护
- 知识文档管理

### 服务索引：`SERVICES_INDEX.md`

所有服务的全局地图，分为三个表：

- **活跃服务**：编号、域名、服务名、服务器、说明
- **归档服务**：已下线服务的历史记录
- **知识文档**：根目录下运维相关的独立文档

### 服务文档：`services/*.md`

每个服务一份文档，基于 `_template/SERVICE.md` 模板填写，包含：

- 基本信息（域名、IP、端口、用途）
- 文件路径（配置、日志、证书位置）
- 配置文件关键项
- Nginx 反代配置要点
- SSL 证书管理
- 常用命令速查
- 运维变更记录

## 配套工具

本框架的 Codex 协作依赖下面两个仓库，要装在运行 Claude Code 的服务器上：

| 工具 | 仓库 | 作用 |
|------|------|------|
| codexmcp | https://github.com/123aliez/codexmcp | MCP 服务，让 Claude Code 调用本机的 Codex CLI；支持并行调用、超时，返回改动文件和执行命令等证据 |
| codex-delegate | https://github.com/123aliez/codex-delegate | Claude Code skill，提供任务书模板和审查流程；第 7 节是本框架用到的运维场景 |

安装前先装好 Codex CLI 并登录：`npm i -g @openai/codex && codex login`。

```bash
# 注册 codexmcp（user 级，名称必须是 codex）。需要代理时加 -e HTTPS_PROXY=... -e HTTP_PROXY=...
claude mcp add codex -s user --transport stdio -- uvx --from git+https://github.com/123aliez/codexmcp.git codexmcp

# 安装 skill（私有仓库，需要有访问权限的账号）
git clone https://github.com/123aliez/codex-delegate.git ~/.claude/skills/codex-delegate

# 以后更新 skill
git -C ~/.claude/skills/codex-delegate pull
```

## 快速开始

### 1. 克隆仓库

```bash
git clone https://github.com/123aliez/ai-linux-ops.git
cd ai-linux-ops
```

### 2. 填写环境信息

```bash
cp _template/DESC_ENV.md DESC_ENV.md
```

编辑 `DESC_ENV.md`，填入你的真实服务器信息。该文件已被 `.gitignore` 排除，只保存在本机。

### 3. 填写服务索引

```bash
cp _template/SERVICES_INDEX.md SERVICES_INDEX.md
```

编辑 `SERVICES_INDEX.md`，登记你的服务（同样只保存在本机）。

### 4. 创建服务文档

基于 `_template/SERVICE.md` 模板，在 `services/` 目录下为每个服务创建文档（`services/` 不入库）：

```bash
cp _template/SERVICE.md services/01_your-service.com.md
```

### 5. 启动使用

按上面的"配套工具"装好 codexmcp 和 codex-delegate 后，用 Claude Code 打开项目目录，AI 会自动加载所有上下文：

```bash
claude
```

## 与 [AI Project Demolition](https://github.com/123aliez/AI_Project_Demolition) 的关系

本框架是 **运维版**，`AI Project Demolition` 是 **开发版**，两者平行独立：

| | AI Project Demolition（开发版） | ai-linux-ops（运维版） |
|---|---|---|
| 管理对象 | 开发项目 | 部署服务 |
| 索引文件 | PROJECTS_INDEX.md | SERVICES_INDEX.md |
| 文档模板 | PROJECT.md | SERVICE.md |
| Codex 的角色 | 执行者（写代码、跑测试），另开会话做独立审查 | 只读助手（信息收集、方案审查、完成审查） |
| 典型场景 | 写代码、修 Bug、重构 | 部署服务、排障、配置变更 |

两者通过文档中的"关联项目/关联服务"字段建立引用关系，但不跨框架操作。

---

# English

# AI Linux Ops — AI-Powered Linux Service Operations Management Framework

A **Linux service operations management framework** designed for AI coding assistants (Claude Code / Codex CLI / Gemini CLI, etc.). Turn your AI assistant into an ops partner that knows your services, config locations, and restart procedures — while strictly respecting operational boundaries.

## What is this?

If you manage a Linux server running dozens of public-facing services (API proxies, monitoring dashboards, databases...), you've probably had to explain your entire setup to AI assistants from scratch every time. This framework solves that.

**In one sentence**: Your AI assistant will fully understand your server and service landscape before it even reads your first message.

## Core Principles

- **Context Pre-loading**: AI auto-reads framework files at startup — no manual environment explanation needed
- **Operational Boundaries**: AI strictly follows "confirm before act" — dangerous commands require human review
- **Tool-Agnostic**: Pure Markdown files, works with Claude Code / Codex / Gemini / Cursor
- **Cross-Session Persistence**: All state is stored in files, independent of AI memory systems
- **Read-only Codex Collaboration**: The main agent plans, coordinates and executes. Read-only Codex sessions gather troubleshooting information, and fresh Codex sessions independently review change plans and results
- **Only the Framework Is Committed**: Real server details and service docs stay on the machine (excluded by `.gitignore`); the repository stays a generic template

## Directory Structure

```
ai-linux-ops/
├── CLAUDE.md              # Claude Code entry (@-imports the rule files)
├── AGENTS.md              # Codex entry (with read-only assistant mode)
├── RULES_GLOBAL.md        # Behavior rules (7 rules)
├── DESC_DIR.md            # Directory management rules (7 rules)
├── .claude/settings.json  # Permissions: dangerous commands always ask
├── .gitignore             # Excludes the local state files below
├── _template/
│   ├── SERVICE.md         # Service document template
│   ├── PROGRESS.md        # Progress tracking template
│   ├── DESC_ENV.md        # Server environment template
│   └── SERVICES_INDEX.md  # Service index template
│
│  Local state, never committed:
├── DESC_ENV.md            # Server environment (IP, OS, software stack)
├── SERVICES_INDEX.md      # Global service index (map of all services)
├── services/              # Service documents directory
└── *.md                   # Knowledge documents
```

## File Reference

### Entry File: `CLAUDE.md`

The entry point auto-loaded by AI assistants. `CLAUDE.md` `@`-imports the four core files. `AGENTS.md` tells Codex to read them in order, and switches it to read-only assistant mode when Claude calls it with a prompt starting with 「【任务书】」. Different AI tools use different filenames:

| AI Tool | Entry File |
|---------|-----------|
| Claude Code | `CLAUDE.md` |
| Codex CLI | `AGENTS.md` |
| Gemini CLI | `GEMINI.md` |

For Gemini CLI, copy `AGENTS.md` and rename it to `GEMINI.md`.

### Behavior Rules: `RULES_GLOBAL.md`

Defines 7 behavioral rules for AI assistants:

| # | Rule | Description |
|---|------|-------------|
| 1 | Language | Respond in user's preferred language |
| 2 | Confirm Before Action | File/server operations require prior confirmation |
| 3 | Context Priority | Parse user-provided documents first, don't use training data |
| 4 | Handshake Protocol | One-sentence confirmation that the framework is loaded |
| 5 | Single Source of Truth | Only write to framework files, never to tool-specific configs |
| 6 | Ops Preferences | Read-only Codex info gathering and independent review, dangerous command review (enforced by ask rules), post-change sync |
| 7 | Network Proxy Fallback | Auto-retry through proxy on network failures |

### Server Environment: `DESC_ENV.md`

Fill in your real server information:

- Server inventory (IP, OS, CPU/RAM, purpose)
- SSH access methods
- Common software stack versions
- Network architecture (reverse proxy, SSL, port planning)
- Working directory conventions

### Directory Rules: `DESC_DIR.md`

7 directory management rules, including:

- Global index first-read policy
- Service context locking (confirm before operating on a specific service)
- New service creation flow (confirm → template → fill → write)
- Proactive document update prompts after changes
- Operational boundary red lines
- Template read-only protection
- Knowledge document management

### Service Index: `SERVICES_INDEX.md`

The global map of all services, organized in three tables:

- **Active Services**: ID, domain, service name, server, description
- **Archived Services**: Historical records of decommissioned services
- **Knowledge Documents**: Standalone ops-related documents at root level

### Service Documents: `services/*.md`

One document per service, based on `_template/SERVICE.md` template, including:

- Basic info (domain, IP, port, purpose)
- File paths (config, logs, certificate locations)
- Configuration key items
- Nginx reverse proxy configuration highlights
- SSL certificate management
- Common command quick reference
- Operational change log

## Companion Tools

The Codex collaboration in this framework relies on two repositories, installed on the server that runs Claude Code:

| Tool | Repository | Purpose |
|------|------------|---------|
| codexmcp | https://github.com/123aliez/codexmcp | MCP server that lets Claude Code call the local Codex CLI; supports parallel calls and timeouts, and returns evidence such as changed files and executed commands |
| codex-delegate | https://github.com/123aliez/codex-delegate | Claude Code skill with task-spec templates and review procedures; section 7 covers the ops scenario this framework uses |

Install and log in to Codex CLI first: `npm i -g @openai/codex && codex login`.

```bash
# Register codexmcp (user scope; the name must be codex). Add -e HTTPS_PROXY=... -e HTTP_PROXY=... if you need a proxy
claude mcp add codex -s user --transport stdio -- uvx --from git+https://github.com/123aliez/codexmcp.git codexmcp

# Install the skill (private repository; needs an account with access)
git clone https://github.com/123aliez/codex-delegate.git ~/.claude/skills/codex-delegate

# Update the skill later
git -C ~/.claude/skills/codex-delegate pull
```

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/123aliez/ai-linux-ops.git
cd ai-linux-ops
```

### 2. Fill in Environment Info

```bash
cp _template/DESC_ENV.md DESC_ENV.md
```

Edit `DESC_ENV.md` with your real server information. The file is excluded by `.gitignore` and stays on the machine.

### 3. Fill in Service Index

```bash
cp _template/SERVICES_INDEX.md SERVICES_INDEX.md
```

Edit `SERVICES_INDEX.md` to register your services (also local only).

### 4. Create Service Documents

Use `_template/SERVICE.md` as a template to create documents in `services/` (not committed):

```bash
cp _template/SERVICE.md services/01_your-service.com.md
```

### 5. Start Using

After installing codexmcp and codex-delegate as described in Companion Tools, open the project directory with Claude Code — AI will auto-load all context:

```bash
claude
```

## Relationship with [AI Project Demolition](https://github.com/123aliez/AI_Project_Demolition)

This framework is the **ops edition**, while `AI Project Demolition` is the **dev edition**. They run in parallel, independently:

| | AI Project Demolition (Dev) | ai-linux-ops (Ops) |
|---|---|---|
| Managed objects | Development projects | Deployed services |
| Index file | PROJECTS_INDEX.md | SERVICES_INDEX.md |
| Doc template | PROJECT.md | SERVICE.md |
| Codex's role | Executor (writes code, runs tests), with a separate review session | Read-only assistant (information gathering, plan review, result review) |
| Typical use cases | Writing code, fixing bugs, refactoring | Deploying services, troubleshooting, config changes |

They reference each other through "related project/service" fields in documents but never cross-operate.

## License

MIT License
