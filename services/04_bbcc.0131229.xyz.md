# 04. CC-Web（bbcc.0131229.xyz）

> Claude Code / Codex 轻量级 Web 远程工具 — 在浏览器中与本机 CLI Agent 交互
> 服务器：154.36.173.87

---

## 基本信息

| 项目 | 值 |
|------|-----|
| 域名 | bbcc.0131229.xyz |
| 服务器 IP | 154.36.173.87 |
| 后端端口 | 8002 |
| 用途 | Claude Code / Codex Web 远程交互界面 |
| 运行方式 | Systemd (Node.js) |
| 项目地址 | https://github.com/ZgDaniel/cc-web |
| 版本 | v1.3.1 |

---

## 文件路径

| 类型 | 路径 |
|------|------|
| 服务目录 | /opt/cc-web |
| 环境配置 | /opt/cc-web/.env |
| 认证配置 | /opt/cc-web/config/auth.json |
| 通知配置 | /opt/cc-web/config/notify.json |
| 会话数据 | /opt/cc-web/sessions/ |
| 日志目录 | /opt/cc-web/logs/ |
| Nginx 配置 | /usr/local/nginx/conf/conf.d/bbcc.0131229.xyz.conf |
| SSL 证书 | /etc/letsencrypt/live/bbcc.0131229.xyz/ |
| Systemd 服务 | /etc/systemd/system/cc-web.service |

---

## 配置文件

### .env 关键项

| 变量 | 值 | 说明 |
|------|-----|------|
| PORT | 8002 | 服务监听端口 |
| CLAUDE_PATH | claude | Claude CLI 路径（/usr/local/bin/claude） |
| CODEX_PATH | codex | Codex CLI 路径（未安装） |

### Systemd 要点

- `KillMode=process` — 只杀 Node.js 进程，不杀 Claude 子进程
- `User=hanna`
- `ExecStart=/usr/local/bin/node server.js`

---

## Nginx 配置要点

- HTTP → HTTPS 301 重定向
- WebSocket 支持（`Upgrade` / `Connection` 头）
- 长连接超时 3600s（适配 Claude 长时间任务）
- SSE 流式响应（`proxy_buffering off`）
- SSL 证书来自 Let's Encrypt（certbot webroot）

---

## SSL 证书

签发方式：Let's Encrypt (certbot webroot)

```bash
# 手动续期
sudo certbot renew --cert-name bbcc.0131229.xyz

# 查看到期时间
sudo certbot certificates --cert-name bbcc.0131229.xyz
```

到期日期：2026-09-14（certbot 已配置自动续期）

---

## 常用命令

```bash
# 服务状态
sudo systemctl status cc-web

# 重启服务
sudo systemctl restart cc-web

# 查看日志
journalctl -u cc-web -f
tail -f /opt/cc-web/logs/process.log | jq .

# 健康检查
curl -sI https://bbcc.0131229.xyz

# Nginx 重载
sudo /usr/local/nginx/sbin/nginx -s reload

# 查看自动生成的密码
cat /opt/cc-web/config/auth.json
```

---

## 运维变更记录

| 日期 | 说明 |
|------|------|
| 2026-06-16 | 初始部署：clone cc-web v1.3.1，systemd + Nginx 反代 + Let's Encrypt SSL |

---

**最后更新**: 2026-06-16
