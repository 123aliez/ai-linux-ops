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
- **Subagent 委派**：主代理做规划和调度，具体操作委派子代理执行
- **Codex 协作审核**：方案设计和任务完成时调用 Codex MCP 交叉验证

## 目录结构

```
ai-linux-ops/
├── CLAUDE.md              # 入口文件（AI 启动时自动读取）
├── RULES_GLOBAL.md        # 行为规则（7 条）
├── DESC_ENV.md            # 服务器环境描述（IP、系统、软件栈）
├── DESC_DIR.md            # 目录管理规范（7 条）
├── SERVICES_INDEX.md      # 全局服务索引（所有服务的地图）
├── _template/
│   ├── SERVICE.md         # 服务文档模板
│   └── PROGRESS.md        # 进度追踪模板
└── services/              # 服务文档目录
    ├── 01_example.com.md
    ├── 02_another.com.md
    └── ...
```

## 文件说明

### 入口文件：`CLAUDE.md`

AI 助手启动时自动读取的入口，指令它按顺序加载四个核心文件。不同 AI 工具对应不同的入口文件名：

| AI 工具 | 入口文件 |
|---------|---------|
| Claude Code | `CLAUDE.md` |
| Codex CLI | `AGENTS.md` |
| Gemini CLI | `GEMINI.md` |

内容相同，只是文件名不同。从模板复制并重命名即可。

### 行为规则：`RULES_GLOBAL.md`

定义 AI 助手的 7 条行为准则：

| # | 规则 | 说明 |
|---|------|------|
| 1 | 语言设定 | 按用户偏好语言回答 |
| 2 | 行动前置确认 | 涉及文件/服务器操作必须先确认 |
| 3 | 上下文解析优先 | 优先读取用户提供的文档，不用预训练数据 |
| 4 | 状态确认握手 | 每次新会话总结已加载的上下文 |
| 5 | 状态存储唯一性 | 只写入本框架的文件，不写工具专属配置 |
| 6 | 运维偏好 | Subagent 委派、Codex 协作审核、高危命令确认、变更后同步 |
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

## 快速开始

### 1. 克隆仓库

```bash
git clone https://github.com/YOUR_USERNAME/ai-linux-ops.git
cd ai-linux-ops
```

### 2. 填写环境信息

编辑 `DESC_ENV.md`，填入你的真实服务器信息。

### 3. 填写服务索引

编辑 `SERVICES_INDEX.md`，登记你的服务。

### 4. 创建服务文档

基于 `_template/SERVICE.md` 模板，在 `services/` 目录下为每个服务创建文档：

```bash
cp _template/SERVICE.md services/01_your-service.com.md
```

### 5. 启动使用

用 Claude Code 打开项目目录，AI 会自动加载所有上下文：

```bash
claude
```

## 与 [ai-project-manager](https://github.com/YOUR_USERNAME/ai-project-manager) 的关系

本框架是 **运维版**，`ai-project-manager` 是 **开发版**，两者平行独立：

| | ai-project-manager（开发版） | ai-linux-ops（运维版） |
|---|---|---|
| 管理对象 | 开发项目 | 部署服务 |
| 索引文件 | PROJECTS_INDEX.md | SERVICES_INDEX.md |
| 文档模板 | PROJECT.md | SERVICE.md |
| 核心偏好 | 开发（subagent 并行开发） | 运维（高危命令确认） |
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
- **Subagent Delegation**: Main agent plans and coordinates; subagents execute specific tasks
- **Codex Collaboration**: Design reviews and task completion verification via Codex MCP

## Directory Structure

```
ai-linux-ops/
├── CLAUDE.md              # Entry file (auto-loaded by AI on startup)
├── RULES_GLOBAL.md        # Behavior rules (7 rules)
├── DESC_ENV.md            # Server environment (IP, OS, software stack)
├── DESC_DIR.md            # Directory management rules (7 rules)
├── SERVICES_INDEX.md      # Global service index (map of all services)
├── _template/
│   ├── SERVICE.md         # Service document template
│   └── PROGRESS.md        # Progress tracking template
└── services/              # Service documents directory
    ├── 01_example.com.md
    ├── 02_another.com.md
    └── ...
```

## File Reference

### Entry File: `CLAUDE.md`

The entry point auto-loaded by AI assistants, instructing them to load four core files in order. Different AI tools use different filenames:

| AI Tool | Entry File |
|---------|-----------|
| Claude Code | `CLAUDE.md` |
| Codex CLI | `AGENTS.md` |
| Gemini CLI | `GEMINI.md` |

Same content, just different filenames. Copy from template and rename.

### Behavior Rules: `RULES_GLOBAL.md`

Defines 7 behavioral rules for AI assistants:

| # | Rule | Description |
|---|------|-------------|
| 1 | Language | Respond in user's preferred language |
| 2 | Confirm Before Action | File/server operations require prior confirmation |
| 3 | Context Priority | Parse user-provided documents first, don't use training data |
| 4 | Handshake Protocol | Summarize loaded context at each new session start |
| 5 | Single Source of Truth | Only write to framework files, never to tool-specific configs |
| 6 | Ops Preferences | Subagent delegation, Codex collaboration, dangerous command review, post-change sync |
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

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/YOUR_USERNAME/ai-linux-ops.git
cd ai-linux-ops
```

### 2. Fill in Environment Info

Edit `DESC_ENV.md` with your real server information.

### 3. Fill in Service Index

Edit `SERVICES_INDEX.md` to register your services.

### 4. Create Service Documents

Use `_template/SERVICE.md` as a template to create documents in `services/`:

```bash
cp _template/SERVICE.md services/01_your-service.com.md
```

### 5. Start Using

Open the project directory with Claude Code — AI will auto-load all context:

```bash
claude
```

## Relationship with [ai-project-manager](https://github.com/YOUR_USERNAME/ai-project-manager)

This framework is the **ops edition**, while `ai-project-manager` is the **dev edition**. They run in parallel, independently:

| | ai-project-manager (Dev) | ai-linux-ops (Ops) |
|---|---|---|
| Managed objects | Development projects | Deployed services |
| Index file | PROJECTS_INDEX.md | SERVICES_INDEX.md |
| Doc template | PROJECT.md | SERVICE.md |
| Core preference | Development (parallel subagent coding) | Operations (dangerous command review) |
| Typical use cases | Writing code, fixing bugs, refactoring | Deploying services, troubleshooting, config changes |

They reference each other through "related project/service" fields in documents but never cross-operate.

## License

MIT License
