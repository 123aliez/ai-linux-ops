# SERVICES_INDEX.md — 全局服务索引

## 活跃服务

| 编号 | 域名 | 服务名 | 服务器 | 说明 |
|------|------|--------|--------|------|
| 01 | bbvpn.hybaliez.com | V2Ray 代理节点 | ser1494860506 (154.36.173.87) | VMess + WS + TLS，Nginx 反代 + 伪装页面 |
| 02 | 本地服务 | Hermes Agent | ser1494860506 (154.36.173.87) | 自学习 AI Agent 平台，微信/Telegram 集成（v0.16.0 / v2026.6.5） |
| 03 | 612.hybaliez.xyz | New-API | ser1494860506 (154.36.173.87) | 大模型网关与 AI 资产管理系统，Docker Compose（new-api+PG+Redis）+ Nginx 反代 |
| 04 | bbcc.0131229.xyz | CC-Web | ser1494860506 (154.36.173.87) | Claude Code / Codex Web 远程交互界面，Node.js + Systemd + Nginx 反代 |
| 05 | huanglian.xyz | 黄连智能分析 | ser1494860506 (154.36.173.87) | 黄连样本质量评估与产地溯源 RAG Agent，FastAPI + LangGraph + ChromaDB，Systemd + Nginx 反代 |

## 归档服务（archive/）

| 编号 | 域名 | 下线日期 | 说明 |
|------|------|---------|------|

## 知识文档

| 文件 | 主题 | 关联服务 | 说明 |
|------|------|---------|------|
| 04_hermes_customization.md | Hermes 个性化加固 | 02_local-hermes | systemd override（内存兜底/OOM 隔离）+ weekly-vacuum.sh 周维护，独立于 hermes update |
| huanglian-deployment.md | 黄连智能分析部署 | 05_huanglian.xyz | systemd 服务配置 + Nginx 反代 + SSL 证书 + 备份恢复流程 |
| huanglian-api-docs.md | 黄连智能分析 API | 05_huanglian.xyz | 完整 API 接口文档（15 个接口） |
