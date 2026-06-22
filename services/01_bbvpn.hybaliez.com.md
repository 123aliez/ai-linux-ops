# 01. V2Ray 代理节点（bbvpn.hybaliez.com）

> VMess + WebSocket + TLS 代理节点，Nginx 反代 + 静态伪装
> 服务器：38.246.232.195

---

## 基本信息

| 项目 | 值 |
|------|-----|
| 域名 | bbvpn.hybaliez.com |
| 服务器 IP | 38.246.232.195 |
| 后端端口 | 10000（V2Ray，仅监听 127.0.0.1） |
| 用途 | VMess 代理节点 |
| 运行方式 | Systemd |
| 项目地址 | /home/hanna/vps_deployment_ai_tools/1.v2ray/ |

---

## 连接参数

| 参数 | 值 |
|------|-----|
| 协议 | VMess |
| 地址 | bbvpn.hybaliez.com |
| 端口 | 443 |
| UUID | c8c00554-1009-452d-a377-020e555a989d |
| alterId | 0 |
| 传输 | WebSocket (ws) |
| 路径 | /ws-ihwl2kfa |
| TLS | 开启 |
| SNI | bbvpn.hybaliez.com |

---

## 文件路径

| 类型 | 路径 |
|------|------|
| V2Ray 配置 | /usr/local/etc/v2ray/config.json |
| V2Ray 日志 | /var/log/v2ray/ |
| Nginx 站点配置 | /usr/local/nginx/conf/conf.d/bbvpn.hybaliez.com.conf |
| SSL 证书 | /usr/local/nginx/conf/ssl/bbvpn.hybaliez.com/ |
| acme.sh 证书 | /root/.acme.sh/bbvpn.hybaliez.com_ecc/ |
| 伪装网站 | /var/www/static/index.html |
| 部署脚本 | /home/hanna/vps_deployment_ai_tools/1.v2ray/ |
| 连接信息 | /home/hanna/vps_deployment_ai_tools/1.v2ray/v2ray_node_info.txt |

---

## Nginx 配置要点

- HTTP (80) 强制跳转 HTTPS (443)
- HTTPS 未启用 HTTP/2（避免 WebSocket 兼容问题）
- 根路径 `/` 返回伪装静态页面
- WebSocket 路径 `/ws-ihwl2kfa` 代理至 V2Ray 127.0.0.1:10000
- 已配置 Cloudflare 真实 IP 还原（set_real_ip_from）
- SSL 协议：TLSv1.2 / TLSv1.3

---

## SSL 证书

签发方式：ZeroSSL（通过 acme.sh 自动申请，ECC 256）

- 域名：bbvpn.hybaliez.com
- 类型：ECC 256
- 到期：2026-09-08
- 自动续期：已配置（acme.sh renew + nginx reload）

```bash
# 查看到期时间
openssl x509 -in /usr/local/nginx/conf/ssl/bbvpn.hybaliez.com/fullchain.pem -noout -dates

# 手动续期
~/.acme.sh/acme.sh --renew -d bbvpn.hybaliez.com --ecc --force
systemctl reload nginx
```

---

## 常用命令

```bash
# V2Ray 服务状态
systemctl status v2ray

# 重启 V2Ray
systemctl restart v2ray

# 查看 V2Ray 日志
journalctl -u v2ray -f

# Nginx 测试配置
/usr/local/nginx/sbin/nginx -t

# 重载 Nginx
systemctl reload nginx

# 查看连接信息
cat /home/hanna/vps_deployment_ai_tools/1.v2ray/v2ray_node_info.txt
```

---

## 已知问题与修复

| 问题 | 原因 | 修复 |
|------|------|------|
| WebSocket 返回 400 | 脚本生成 `http2 on;` 导致 h2 协商，WS 不兼容 | 移除 Nginx 配置中 `http2 on;` 行 |

> **注意**：每次重跑 `install_v2ray.sh` 后需手动移除 `http2 on;` 并 reload nginx。

---

## 运维变更记录

| 日期 | 说明 |
|------|------|
| 2026-06-10 | 初始部署：Nginx 1.28.1 + V2Ray 5.49.0 + ZeroSSL ECC + 伪装页面 |
| 2026-06-10 | 重新部署：清理旧配置重跑脚本，修复 http2 兼容问题 |

---

**最后更新**: 2026-06-10
