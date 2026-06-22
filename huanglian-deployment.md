# 黄连智能分析系统 - 部署运维文档

## 项目信息

- **项目名称**: 黄连智能分析 Agent
- **域名**: huanglian.xyz
- **服务器**: 154.36.173.87
- **部署路径**: /home/hanna/Project/huanglian
- **部署时间**: 2026-06-17
- **SSL 证书**: Let's Encrypt (有效期至 2026-09-15，自动续期)

## 技术栈

### 后端
- FastAPI + Pydantic v2
- LangGraph Agent
- ChromaDB (RAG)
- Python 3.12

### API 配置
- **聊天模型**: ai98pro.xyz (OpenAI 兼容接口)
  - 模型: gpt-5.4-mini
  - Base URL: https://ai98pro.xyz
- **Embedding 模型**: DashScope (阿里云原生接口)
  - 模型: text-embedding-v4
  - Base URL: https://dashscope-us.aliyuncs.com

### 反向代理
- Nginx (自建编译版本)
- Let's Encrypt SSL 证书（自动续期）

## 目录结构

```
/home/hanna/Project/huanglian/
├── app/                    # 后端代码
│   ├── api/               # FastAPI 路由
│   ├── agent/             # LangGraph Agent
│   ├── services/          # 业务服务层
│   └── tools/             # 工具层
├── data/huanglian/        # 数据目录
├── chroma_db/             # ChromaDB 向量库
├── logs/                  # 日志
├── .env                   # 环境变量（不提交 git）
└── .venv/                 # Python 虚拟环境
```

## 环境变量

关键环境变量配置在 `.env` 文件中：

```bash
# 聊天模型 (OpenAI 兼容)
OPENAI_API_KEY=sk-xxx...
OPENAI_BASE_URL=https://ai98pro.xyz

# Embedding 模型 (DashScope)
DASHSCOPE_API_KEY=sk-ws-xxx...

# 服务配置
API_HOST=127.0.0.1
API_PORT=8000

# 模型配置
CHAT_MODEL_NAME=gpt-5.4-mini
EMBEDDING_MODEL_NAME=text-embedding-v4
```

## 服务管理

### 1. 启动后端服务

```bash
cd /home/hanna/Project/huanglian

# 激活虚拟环境
source .venv/bin/activate

# 启动 FastAPI 服务
uvicorn app.api.main:app --host 127.0.0.1 --port 8000
```

### 2. 后台运行（使用 systemd）

创建服务文件 `/etc/systemd/system/huanglian.service`:

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

服务命令：
```bash
sudo systemctl daemon-reload
sudo systemctl enable huanglian
sudo systemctl start huanglian
sudo systemctl status huanglian
sudo systemctl stop huanglian
sudo systemctl restart huanglian
```

### 3. Nginx 反向代理配置

配置文件 `/usr/local/nginx/conf/conf.d/huanglian.xyz.conf`:

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name huanglian.xyz;

    # SSL 证书配置
    ssl_certificate /usr/local/nginx/conf/ssl/huanglian.xyz/fullchain.pem;
    ssl_certificate_key /usr/local/nginx/conf/ssl/huanglian.xyz/privkey.pem;

    # SSL 安全配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # 访问日志
    access_log /var/log/nginx/huanglian.access.log;
    error_log /var/log/nginx/huanglian.error.log;

    # CORS 配置
    add_header 'Access-Control-Allow-Origin' '*' always;
    add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
    add_header 'Access-Control-Allow-Headers' 'Content-Type, Authorization' always;

    # OPTIONS 预检请求
    if ($request_method = 'OPTIONS') {
        return 204;
    }

    # 反向代理到本地 FastAPI 服务
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket 支持
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # 超时配置
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}

# HTTP 自动跳转到 HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name huanglian.xyz;
    return 301 https://$server_name$request_uri;
}
```

Nginx 服务命令：
```bash
sudo systemctl reload nginx
sudo systemctl status nginx
sudo systemctl restart nginx

# 测试配置文件
sudo /usr/local/nginx/sbin/nginx -t
```

## 前端部署

### 构建前端

```bash
cd /home/hanna/Project/huanglian/frontend

# 安装依赖
npm install

# 构建生产版本
npm run build
```

构建产物位于 `frontend/dist/` 目录。

### Nginx 前端配置

Nginx 配置中已包含前端静态文件服务：

```nginx
# 前端静态文件
root /home/hanna/Project/huanglian/frontend/dist;
index index.html;

# 前端路由 - 所有非 API 请求返回 index.html（支持 SPA 路由）
location / {
    try_files $uri $uri/ /index.html;
}

# API 反向代理
location /api/ {
    proxy_pass http://127.0.0.1:8000;
    # ... 其他配置
}
```

### 访问地址

- 前端界面：https://huanglian.xyz/
- 健康检查：https://huanglian.xyz/api/health
- 聊天接口：POST https://huanglian.xyz/api/chat

### 前端更新流程

```bash
cd /home/hanna/Project/huanglian/frontend

# 1. 拉取最新代码
git pull origin main

# 2. 安装依赖（如有变化）
npm install

# 3. 构建
npm run build

# 4. Nginx 自动使用新的静态文件，无需重启
```

---

## 后端 API 路径

所有 API 路径统一使用 `/api` 前缀：

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 健康检查 | GET | /api/health | 服务状态检查 |
| 聊天问答 | POST | /api/chat | 完整响应（等待全部生成） |
| 流式聊天 | POST | /api/chat/stream | SSE 流式输出（实时显示） |
| 质量评估 | POST | /api/quality/predict | 样本质量评估 |
| 产地溯源 | POST | /api/traceability/predict | 样本产地溯源 |
| 生成报告 | POST | /api/report/generate | 生成分析报告 |
| 报告详情 | GET | /api/report/{report_id} | 获取已保存的报告 |

### 流式输出接口说明

流式聊天接口使用 Server-Sent Events (SSE) 协议：

**请求**：
```bash
curl -X POST https://huanglian.xyz/api/chat/stream \
  -H "Content-Type: application/json" \
  -d '{"query": "什么是黄连？"}'
```

**响应格式**：
```
data: {"type": "start", "query": "什么是黄连？"}

data: {"type": "chunk", "content": "一"}

data: {"type": "chunk", "content": "、"}

data: {"type": "chunk", "content": "结论"}

data: {"type": "end", "answer": "完整回答内容..."}
```

**事件类型**：
- `start` - 流式开始
- `chunk` - 文本片段
- `end` - 流式结束
- `error` - 错误信息

---

### 1. 本地测试

```bash
cd /home/hanna/Project/huanglian

# 运行完整测试套件
DASHSCOPE_API_KEY="your-key" .venv/bin/python -m pytest tests/ -v

# 运行集成测试
.venv/bin/python -m pytest tests/integration/ -v

# 运行单元测试
.venv/bin/python -m pytest tests/unit/ -v
```

### 2. API 健康检查

```bash
# 本地检查
curl http://localhost:8000/health

# 域名检查
curl https://huanglian.xyz/health
```

预期响应：
```json
{
  "data": {
    "status": "ok"
  }
}
```

### 3. 聊天接口测试

```bash
curl -X POST https://huanglian.xyz/chat \
  -H "Content-Type: application/json" \
  -d '{"query": "黄连的质量评估指标有哪些？"}'
```

## 日志管理

### 应用日志
- 位置: `/home/hanna/Project/huanglian/logs/`
- 查看: `tail -f logs/app.log`

### Nginx 访问日志
- 位置: `/var/log/nginx/huanglian.access.log` 和 `/var/log/nginx/huanglian.error.log`
- 查看: `sudo tail -f /var/log/nginx/huanglian.access.log`

### 系统日志
```bash
# 查看后端服务日志
sudo journalctl -u huanglian -f

# 查看 Nginx 日志
sudo journalctl -u nginx -f
```

## 故障排查

### 1. 服务无法启动

检查端口占用：
```bash
sudo lsof -i :8000
```

检查环境变量：
```bash
cd /home/hanna/Project/huanglian
source .venv/bin/activate
python -c "import os; from dotenv import load_dotenv; load_dotenv(); print('Keys OK' if os.getenv('OPENAI_API_KEY') and os.getenv('DASHSCOPE_API_KEY') else 'Keys Missing')"
```

### 2. SSL 证书管理

**当前证书信息**：
```bash
# 查看证书详情
sudo certbot certificates

# 证书位置
# /etc/letsencrypt/live/huanglian.xyz/fullchain.pem
# /etc/letsencrypt/live/huanglian.xyz/privkey.pem
```

**证书自动续期**：
Certbot 已配置自动续期任务，证书到期前会自动更新。手动测试续期：
```bash
# 测试续期（不实际执行）
sudo certbot renew --dry-run

# 手动强制续期
sudo certbot renew --force-renewal
```

**重新申请证书**（如需更换域名或证书损坏）：
```bash
# 停止 Nginx
sudo systemctl stop nginx

# 申请新证书
sudo certbot certonly --standalone -d huanglian.xyz --force-renewal

# 启动 Nginx
sudo systemctl start nginx
```

### 3. API 调用失败

检查 API Key 有效性：
```bash
# 测试 DashScope
curl -X POST "https://dashscope-us.aliyuncs.com/api/v1/services/embeddings/text-embedding/text-embedding" \
  -H "Authorization: Bearer $DASHSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "text-embedding-v4", "input": {"texts": ["test"]}}'

# 测试聊天模型
curl -X POST "https://ai98pro.xyz/v1/chat/completions" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.4-mini", "messages": [{"role": "user", "content": "test"}]}'
```

## 备份与恢复

### 备份清单

1. **代码**: Git 仓库（已提交）
2. **环境变量**: `.env` 文件（手动备份）
3. **ChromaDB 数据**: `chroma_db/` 目录
4. **上传文件**: `data/huanglian/uploads/`
5. **生成报告**: `data/huanglian/reports/generated/`

### 备份命令

```bash
# 创建备份目录
mkdir -p /home/hanna/backups/huanglian-$(date +%Y%m%d)

# 备份数据
cd /home/hanna/Project/huanglian
tar czf /home/hanna/backups/huanglian-$(date +%Y%m%d)/data.tar.gz data/
tar czf /home/hanna/backups/huanglian-$(date +%Y%m%d)/chroma_db.tar.gz chroma_db/
cp .env /home/hanna/backups/huanglian-$(date +%Y%m%d)/.env.backup
```

### 恢复命令

```bash
cd /home/hanna/Project/huanglian

# 恢复数据
tar xzf /home/hanna/backups/huanglian-YYYYMMDD/data.tar.gz
tar xzf /home/hanna/backups/huanglian-YYYYMMDD/chroma_db.tar.gz

# 恢复环境变量
cp /home/hanna/backups/huanglian-YYYYMMDD/.env.backup .env

# 重启服务
sudo systemctl restart huanglian
```

## 更新部署

### 代码更新流程

```bash
cd /home/hanna/Project/huanglian

# 1. 拉取最新代码
git pull origin main

# 2. 更新依赖（如有变化）
source .venv/bin/activate
uv pip install -r pyproject.toml

# 3. 运行测试
DASHSCOPE_API_KEY="your-key" .venv/bin/python -m pytest tests/ -v

# 4. 重启服务
sudo systemctl restart huanglian

# 5. 验证部署
curl https://huanglian.xyz/health
```

## 监控指标

### 关键指标

1. **服务可用性**: 健康检查响应时间
2. **API 调用成功率**: 200 状态码比例
3. **响应时间**: P50/P95/P99 延迟
4. **错误率**: 4xx/5xx 错误比例
5. **资源使用**: CPU/内存/磁盘

### 监控命令

```bash
# 检查服务状态
sudo systemctl status huanglian

# 检查资源使用
htop
df -h
free -h

# 检查进程
ps aux | grep uvicorn
```

## 安全建议

1. **API Key 保护**: 
   - `.env` 文件权限设为 600
   - 不要提交到 Git
   - 定期轮换

2. **防火墙配置**:
   ```bash
   sudo ufw allow 80/tcp
   sudo ufw allow 443/tcp
   sudo ufw enable
   ```

3. **定期更新**:
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

4. **日志审计**: 定期检查异常访问

## 联系方式

- **项目路径**: /home/hanna/Project/huanglian
- **Git 仓库**: 本地仓库
- **部署日期**: 2026-06-17
- **文档位置**: /home/hanna/ai-linux-ops/huanglian-deployment.md
