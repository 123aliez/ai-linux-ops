# 05. 黄连智能分析系统（huanglian.xyz）

> 黄连样本质量评估与产地溯源 RAG Agent（FastAPI + LangGraph + ChromaDB）
> 服务器：154.36.173.87

---

## 基本信息

| 项目 | 值 |
|------|-----|
| 域名 | huanglian.xyz |
| 服务器 IP | 154.36.173.87 |
| 后端端口 | 8000（uvicorn，仅监听 127.0.0.1） |
| 用途 | 黄连样本质量评估 / 产地溯源 / RAG 问答 / 报告生成 |
| 运行方式 | Systemd（uvicorn）+ Nginx 反代 |
| 项目地址 | /home/hanna/Project/huanglian |
| Git 仓库 | github.com:123aliez/huanglian.git |

---

## 文件路径

| 类型 | 路径 |
|------|------|
| 项目目录 | /home/hanna/Project/huanglian |
| 环境变量 | /home/hanna/Project/huanglian/.env |
| Systemd 服务 | /etc/systemd/system/huanglian.service |
| Nginx 站点配置 | /usr/local/nginx/conf/conf.d/huanglian.xyz.conf |
| SSL 证书 | /usr/local/nginx/conf/ssl/huanglian.xyz/ |
| 前端构建产物 | /home/hanna/Project/huanglian/frontend/dist/ |
| ChromaDB 数据 | /home/hanna/Project/huanglian/chroma_db/ |
| 应用日志 | /home/hanna/Project/huanglian/logs/ |

---

## 配置文件

### Systemd 服务（huanglian.service）

```ini
[Unit]
Description=Huanglian Agent API
After=network.target

[Service]
Type=simple
User=hanna
WorkingDirectory=/home/hanna/Project/huanglian
Environment="PATH=/home/hanna/Project/huanglian/.venv/bin"
ExecStart=/home/hanna/Project/huanglian/.venv/bin/uvicorn app.api.main:app --host 127.0.0.1 --port 8000
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### .env 关键项

```bash
OPENAI_API_KEY=sk-xxx          # 聊天模型（ai98pro.xyz, gpt-5.4-mini）
OPENAI_BASE_URL=https://ai98pro.xyz
DASHSCOPE_API_KEY=sk-ws-xxx    # Embedding（DashScope, text-embedding-v4）
API_HOST=127.0.0.1
API_PORT=8000
CHAT_MODEL_NAME=gpt-5.4-mini
EMBEDDING_MODEL_NAME=text-embedding-v4
```

---

## Nginx 配置要点

- HTTP (80) 强制跳转 HTTPS (443)，启用 HTTP/2
- 前端静态文件：root `frontend/dist`，SPA 路由 `try_files $uri /index.html`
- API 反代：`/api/` → `http://127.0.0.1:8000`
- CORS：`Access-Control-Allow-Origin *`，OPTIONS 预检返回 204
- WebSocket / SSE 支持（chat/stream）
- 代理超时 60s

---

## SSL 证书

签发方式：Let's Encrypt（certbot，自动续期）

- 域名：huanglian.xyz
- 到期：2026-09-15
- 自动续期：已配置

```bash
# 查看到期时间
sudo certbot certificates

# 测试续期（不实际执行）
sudo certbot renew --dry-run
```

---

## 常用命令

```bash
# 服务状态
sudo systemctl status huanglian

# 重启后端（代码更新后）
sudo systemctl restart huanglian

# 后端日志
sudo journalctl -u huanglian -f

# 健康检查
curl https://huanglian.xyz/api/health
curl http://127.0.0.1:8000/api/health

# Nginx
sudo /usr/local/nginx/sbin/nginx -t
sudo systemctl reload nginx

# 前端更新（构建后 Nginx 自动用新静态文件，无需重启）
cd /home/hanna/Project/huanglian/frontend && git pull && npm install && npm run build
```

---

## 更新部署流程

```bash
cd /home/hanna/Project/huanglian

# 1. 拉取最新代码
git pull origin main

# 2. 更新依赖（如有变化）
source .venv/bin/activate
uv pip install -r pyproject.toml

# 3. 前端构建（如有前端改动）
cd frontend && npm install && npm run build && cd ..

# 4. 重启后端
sudo systemctl restart huanglian

# 5. 验证
curl https://huanglian.xyz/api/health
```

---

## API 接口

所有 API 统一 `/api` 前缀：

| 接口 | 方法 | 路径 |
|------|------|------|
| 健康检查 | GET | /api/health |
| 聊天问答 | POST | /api/chat |
| 流式聊天 | POST | /api/chat/stream（SSE） |
| 质量评估 | POST | /api/quality/predict |
| 产地溯源 | POST | /api/traceability/predict |
| 生成报告 | POST | /api/report/generate |

完整接口文档见 `huanglian-api-docs.md`。

---

## 详细文档

- **部署/运维详解**：[huanglian-deployment.md](../huanglian-deployment.md)（systemd + nginx + SSL + 备份恢复）
- **API 接口文档**：[huanglian-api-docs.md](../huanglian-api-docs.md)
- **项目架构**：/home/hanna/Project/huanglian/PROJECT.md

---

## 运维变更记录

| 日期 | 说明 |
|------|------|
| 2026-06-17 | 初始部署：FastAPI + LangGraph + ChromaDB，Systemd + Nginx + Let's Encrypt SSL |
| 2026-06-18 | 容错架构 + QC 接入 + Agent 路由对齐 + harness real-agent 100%（git: 9973f95 → 59adf5c，5 commit） |
| 2026-06-21 | 前端补全质量/产地/报告独立入口 + image-only 上传修复；huanglian.xyz 前后端全链路可达（git: 2c202f1 + 6c739fe） |

---

**最后更新**: 2026-06-18
