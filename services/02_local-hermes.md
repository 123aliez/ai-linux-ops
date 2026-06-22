# 02. Hermes Agent（本地服务）

> 自学习 AI Agent 平台（Gateway + Dashboard），集成微信、Telegram 等消息平台
> 服务器：ser1494860506 (38.246.232.195)

---

## 基本信息

| 项目 | 值 |
|------|-----|
| 域名 | 无（本地服务） |
| 服务器 IP | 38.246.232.195 |
| 后端端口 | 9119（Dashboard，仅 127.0.0.1） |
| 用途 | AI Agent 自动化平台，自学习技能、消息平台集成 |
| 运行方式 | Systemd (user-level) |
| 项目地址 | https://github.com/nousresearch/hermes-agent (MIT) |
| 版本 | 0.16.0 (v2026.6.5, upstream `4b5ba112a`) |

---

## 文件路径

| 类型 | 路径 |
|------|------|
| 服务目录 | ~/.hermes/hermes-agent/ |
| 配置文件 | ~/.hermes/config.yaml |
| 环境变量 | ~/.hermes/.env |
| Systemd 服务 | ~/.config/systemd/user/hermes-gateway.service |
| | ~/.config/systemd/user/hermes-dashboard.service |
| gateway override | ~/.config/systemd/user/hermes-gateway.service.d/override.conf |
| dashboard override | ~/.config/systemd/user/hermes-dashboard.service.d/override.conf |
| 周维护脚本 | ~/.hermes/scripts/weekly-vacuum.sh（cron `0 4 * * 0`） |

---

## 配置文件

### 模型配置

| 用途 | Provider | 模型 | API 端点 |
|------|----------|------|----------|
| 主控模型 | mimo | mimo-v2.5 | https://token-plan-cn.xiaomimimo.com/v1 |
| 网页提取 | custom | gpt-5.4 | https://newapi.6553601.xyz/v1 |
| 视觉辅助 | custom | mimo-v2.5 | https://token-plan-cn.xiaomimimo.com/v1 |
| 图片生成 | xai | grok-imagine-image-lite | https://jiuuij.de5.net/v1 |
| 搜索 (MCP) | xai | grok-4.20-0309-non-reasoning-console | https://jiuuij.de5.net/v1 |

### 消息平台集成

| 平台 | 数量 | 白名单 | 备注 |
|------|------|--------|------|
| 微信 | 2 个账号 | 已配置 | pairing 模式，群聊禁用 |
| Telegram | 1 个 Bot | 已配置 (7220455655) | |

### Systemd Unit 关键项

- **hermes-gateway**: `Restart=always`，`RestartMaxDelaySec=300`（v0.16.0 主文件已移除该项），`TimeoutStopSec=210`，`KillMode=mixed`（被 override 改为 `control-group`）
- **hermes-dashboard**: `BindsTo=hermes-gateway.service`，随 gateway 自动启停，`Restart=on-failure`
- **内存兜底（override）**：gateway `MemoryHigh=640M / MemoryMax=768M`，dashboard `MemoryHigh=512M / MemoryMax=640M`。详见 `04_hermes_customization.md`。

### 周维护

- `~/.hermes/scripts/weekly-vacuum.sh`，cron 每周日 04:00 UTC：停服务 → 备份 + `hermes sessions optimize`（FTS5 段合并 + VACUUM）→ 截断 mcp-stderr.log → 清理旧备份/sessions → 重启 → 记录 `[METRICS]` cgroup 快照至 `~/.hermes/logs/maintenance.log`。详见 `04_hermes_customization.md` 第 3 节。

---

## 常用命令

```bash
# 服务状态
systemctl --user status hermes-gateway hermes-dashboard

# 重启服务
systemctl --user restart hermes-gateway    # dashboard 会自动跟随重启

# 查看日志
journalctl --user -u hermes-gateway -f     # 实时跟踪
journalctl --user -u hermes-gateway -n 50  # 最近 50 条

# 健康检查
ss -tlnp | grep 9119                       # Dashboard 端口
curl -s http://127.0.0.1:9119/             # Dashboard 页面

# 配置文件
cat ~/.hermes/config.yaml
cat ~/.hermes/.env
```

---

## 已知问题

| 问题 | 严重度 | 说明 |
|------|--------|------|
| mimo API 频繁超时 | 🟡 中 | mimo-v2.5 的 chat completion 请求经常 Request timed out，自动重试后恢复 |
| Credential pool provider mismatch | 🟢 低 | 无害警告，`pool=custom:mimo` 与 `agent=custom` 命名不一致导致，不影响功能 |
| MCP 配置明文 API Key | 🟢 低 | grok-search MCP 的 GROK_API_KEY 直接写在 config.yaml，其他位置已用环境变量 |

---

## 运维变更记录

| 日期 | 说明 |
|------|------|
| 2026-06-11 | Hermes 从原环境迁移至本机，配置 systemd user-level 服务并启动 |
| 2026-06-11 | 健康检查：尝试统一 provider 命名和添加 fallback，认证失败已回滚 |
| 2026-06-14 | `hermes update --yes` 同步上游 1549 commits（`355af2c20` → `4b5ba112a`，v0.15.1 → v0.16.0 / v2026.6.5，落后上游 0）。update 后 dashboard 未自启，已手动拉起。主 service 文件被上游 `--replace` 重写：gateway `ExecStart` 移除 `--replace`、删除 `RestartMaxDelaySec/RestartSteps`；dashboard 无变化。drop-in override 不受影响。 |
| 2026-06-14 | 部署运维加固（详见 `04_hermes_customization.md`）：gateway override（KillMode=control-group / MemoryMax=768M / MemoryHigh=640M / SuccessExitStatus=1 SIGTERM）、dashboard override（MemoryHigh=512M / MemoryMax=640M）、weekly-vacuum.sh + cron（`0 4 * * 0`）。运行时实测全部生效。 |

---

**最后更新**: 2026-06-14
