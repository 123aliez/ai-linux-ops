<desc_env>
# 服务器运行环境

<!-- 由 _template/DESC_ENV.md 复制到工作区根目录后填写。该文件已被 .gitignore 排除，只保存在本机。 -->

## 服务器清单

| 名称 | IP | 系统 | CPU/内存 | 主要用途 | SSH 端口 |
|------|----|------|----------|---------|----------|
| {主机名} | {IP} | {发行版与版本} | {CPU / 内存 / 磁盘} | {用途} | {端口} |

## 通用软件栈

| 软件 | 版本 | 说明 |
|------|------|------|
| {软件} | {版本} | |

## SSH 访问

| 服务器 | 方式 | 用户 | 备注 |
|--------|------|------|------|
| {主机名} | {密钥 / 密码} | {用户} | |

## 网络架构

- **公网 IP**：
- **反向代理**：
- **SSL 管理**：
- **防火墙**：
- **对外端口**：
- **DNS**：

## 工作目录约定

| 用途 | 路径 |
|------|------|
| 用户主目录 | |
| 运维框架 | |
| 部署目录 | |

## AI 工具链

- **Claude Code**：主控，负责规划、调度和执行变更，主模型 Opus。
- **Codex CLI**：只读助手（信息收集、独立审查），经 codexmcp 调用。版本用 `codex --version`，登录状态用 `codex login status` 查询。
- **codexmcp**：`https://github.com/123aliez/codexmcp`，以 user 级 MCP 注册，名称为 `codex`。
- **codex-delegate skill**：`https://github.com/123aliez/codex-delegate`，克隆到 `~/.claude/skills/codex-delegate/`。
</desc_env>
