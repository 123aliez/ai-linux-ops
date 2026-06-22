# Hermes 个性化修改

> 关联服务：02_local-hermes
> 对 Hermes 的本地运维加固，独立于版本升级，`hermes update` 不会覆盖
>
> **实施状态（2026-06-14 实测对齐 ser1494860506/hanna）**：本文档全部 4 项加固（gateway override / dashboard override / weekly-vacuum.sh / cron）已确认在本机落地生效，运行时值见各章节"验证"小节。
> 早期版本中"2026-06-14 已同步上游 `8f278403d`"的记录经核实未在本机生效，已于 2026-06-14 重新同步至 `4b5ba112a`（v0.16.0），详见文末运维变更记录。

---

## 1. 问题背景

Hermes Gateway 运行在 7.8GB 内存的 KVM 虚拟机上，内存泄漏导致 OOM Kill 时会殃及全系统（Docker 容器、Nginx 等），且 MCP 子进程成为僵尸。需要：

1. 将 OOM 爆炸半径限制在 gateway 自身
2. 确保停止/崩溃时 MCP 子进程被完全清理
3. 防止 state.db 和 session 文件无限膨胀

---

## 2. systemd drop-in override

### 为什么用 override 而非直接改 service 文件？

`hermes update` 会通过 `--replace` 参数**重写主 service 文件**。直接修改 `hermes-gateway.service` 会在下次升级时丢失。drop-in override 文件不受影响。

### 文件位置

```
~/.config/systemd/user/hermes-gateway.service.d/override.conf
```

### 内容

```ini
[Service]
KillMode=control-group
MemoryMax=768M
MemoryHigh=640M
SuccessExitStatus=1 SIGTERM
```

### 各配置项说明

| 配置 | 值 | 机制 |
|------|-----|------|
| `KillMode=control-group` | 替换原来的 `mixed` | 停止/OOM 时杀掉 cgroup 内**所有进程**（含 MCP 子进程 arxiv-mcp-serve、codexmcp、zotero-mcp、grok-search），防止僵尸残留 |
| `MemoryHigh=640M` | 软上限 | 超过后 systemd 施加内存压力，促使进程回收；不直接杀，给 gateway 喘息空间 |
| `MemoryMax=768M` | 硬上限 | 超过时 systemd 杀掉 gateway cgroup，触发 Restart=always 自动拉起。**只影响 gateway 本身**，Docker/Nginx/V2Ray 等其他服务不受影响 |
| `SuccessExitStatus=1 SIGTERM` | 把 SIGTERM 引发的 exit=1 视为成功 | weekly-vacuum.sh `systemctl stop` 时 gateway 退出码为 1，systemd 默认会标记 `Failed with result 'exit-code'` 污染 `failed units` 列表。**不影响 `Restart=always` 真崩溃时的拉起逻辑** |

### 与 OOM Kill 的区别

```
OOM Kill（无 MemoryMax）:
  内核选择杀哪个进程 → 可能是 Docker/Nginx/其他关键服务
  → 连锁故障

MemoryMax（有 override）:
  systemd 只杀 gateway cgroup → Restart=always 自动恢复
  → 其他服务完全不受影响
```

### 调整方式

如需修改限制（比如加了更多 MCP 工具后基线上升）：

```bash
# 编辑 override
vi ~/.config/systemd/user/hermes-gateway.service.d/override.conf

# 生效
systemctl --user daemon-reload
systemctl --user restart hermes-gateway
```

### 验证

```bash
# 确认配置生效
systemctl --user show hermes-gateway | grep -E '^MemoryMax=|^MemoryHigh=|^KillMode=|^SuccessExitStatus='
# 预期：MemoryMax=805306368 (768M), MemoryHigh=671088640 (640M), KillMode=control-group, SuccessExitStatus=1 TERM

# 查看当前内存使用
systemctl --user status hermes-gateway | grep Memory
# 预期：Memory: xxxM (high: 640.0M max: 768.0M available: xxxM)
```

---

## 2.1 hermes-dashboard 内存兜底（2026-06-14 新增）

### 背景

v0.16.0 升级时只给 gateway 加了内存上限，dashboard 完全没有兜底（`MemoryMax=infinity`）。BindsTo 关系下 dashboard 自身崩溃不会拖累 gateway，但 dashboard 内存若长期上涨（自身 + 5 个 MCP 子进程在 dashboard cgroup 内）会压缩主机可用内存，与 gateway / SQLite page cache 争抢。

### 文件位置

```
~/.config/systemd/user/hermes-dashboard.service.d/override.conf
```

### 内容

```ini
[Service]
MemoryHigh=512M
MemoryMax=640M
```

### 配置项说明

| 配置 | 值 | 机制 |
|------|-----|------|
| `MemoryHigh=512M` | 软上限 | dashboard cgroup 总内存（含 5 个 MCP 子进程）超过 512M 后施加压力 |
| `MemoryMax=640M` | 硬上限 | 超过 640M 杀 dashboard cgroup，BindsTo 关系下不会触发 gateway 重启；`Restart=on-failure` 自动拉起 |

dashboard 主文件 `Restart=on-failure`，与 gateway 的 `Restart=always` 行为不同——但 OOM 触发的 systemd-kill 退出码非 0，仍属 on-failure 范畴，会自动恢复。

### 调整方式

```bash
vi ~/.config/systemd/user/hermes-dashboard.service.d/override.conf
systemctl --user daemon-reload
systemctl --user restart hermes-dashboard
```

---

## 3. 每周自动维护 cron

### 脚本位置

```
~/.hermes/scripts/weekly-vacuum.sh
```

### 执行时间

```
每周日 04:00 UTC（crontab）
```

### 执行内容

| 步骤 | 操作 | 目的 |
|------|------|------|
| 1 | `systemctl --user stop hermes-gateway` | 获取 state.db 排他锁 |
| 2 | `: > ~/.hermes/logs/mcp-stderr.log` | 截断 MCP stderr 日志（上游无 rotation，30 天约累积 70-100MB）|
| 3 | 备份 state.db | 安全网 |
| 4 | `hermes sessions optimize`（FTS5 段合并 + VACUUM）| 压缩数据库碎片 + 全文索引段合并（比纯 sqlite3 VACUUM 多做一步段合并） |
| 5 | 清理 28 天前的 state.db 备份 | 节省磁盘 |
| 6 | 清理 30 天前的 session 文件 | 控制 sessions/ 目录膨胀 |
| 7 | `systemctl --user start hermes-gateway + dashboard` | 恢复服务 |
| 8 | 记录日志到 `~/.hermes/logs/maintenance.log`（含 `[METRICS]` cgroup 内存快照）| 可追溯，长期形成 RSS 趋势 |

### 影响

- **服务中断**：约 30 秒 ~ 1 分钟（凌晨 4 点低峰时段）
- **Telegram/微信消息**：中断期间的消息会在 gateway 恢复后正常处理
- **state.db**：如果无碎片则大小不变（首次 VACUUM 确认 409MB 全是真实数据）

### 查看维护日志

```bash
cat ~/.hermes/logs/maintenance.log

# 典型一行：
# 2026-06-14 04:00:01 [START] weekly maintenance
# 2026-06-14 04:00:44 [DONE] state.db=414M
# 2026-06-14 04:00:44 [METRICS]
# MemoryCurrent=482344960
# MemoryPeak=496386048
# NRestarts=0
# ActiveEnterTimestamp=Sun 2026-06-14 04:00:41 UTC
```

`[METRICS]` 块替代 v0.16.0 后失效的 `[MEMORY]` 周期日志，长期可看 RSS 趋势。

### 手动触发

```bash
~/.hermes/scripts/weekly-vacuum.sh
```

### 取消 cron

```bash
crontab -e  # 删除 weekly-vacuum.sh 那行
```

---

## 4. 完整 crontab 现状

```
0 0 * * *   /home/user/new-api-backup.sh           # New-API 每日备份
0 4 * * 0   /home/user/.hermes/scripts/weekly-vacuum.sh  # Hermes 每周维护
```

---

## 5. 监控与排障

### 内部 [MEMORY] 周期日志已失效（v0.16.0 起）

v0.15.1 时 `gateway/memory_monitor.py` 通过 `start_memory_monitoring()` 在 `gateway/run.py` 中被调用，每 300s 写一行 `[MEMORY] rss=xxxMB gc=(...) threads=N`。**v0.16.0 重构后 `gateway/run.py` 中已不再 import / 调用 memory_monitor 模块**（文件本身仍保留在仓库内但成为孤儿代码）。

**影响**：失去内部 RSS / GC / threads 时间序列，原文档里 `grep '\[MEMORY\]' ~/.hermes/logs/agent.log` 这套观测手段对 v0.16.0+ 实例无效。

**替代方案**：
1. weekly-vacuum.sh 末尾 `[METRICS]` 块每周抓一次 cgroup 内存快照，长期形成趋势。
2. 实时观测改用 cgroup：`systemctl --user show hermes-gateway -p MemoryCurrent,MemoryPeak,NRestarts`。
3. 如需更高频采样，可手工跑 `watch -n60 'systemctl --user show hermes-gateway -p MemoryCurrent,MemoryPeak'`。

### 确认内存泄漏是否复发

```bash
# v0.16.0 后用 cgroup 替代 [MEMORY] 日志
systemctl --user show hermes-gateway -p MemoryCurrent,MemoryPeak,NRestarts
# 正常：MemoryPeak 稳定在 400-500M，NRestarts=0
# 异常：MemoryPeak 持续逼近 768M 或 NRestarts 递增 → 可能有新泄漏，下调 MemoryMax 或联系上游

# 周维护历史趋势
grep -A4 '\[METRICS\]' ~/.hermes/logs/maintenance.log | tail -30
```

### 确认 MCP 子进程没有泄漏

```bash
# 查看 gateway cgroup 内所有进程
systemctl --user status hermes-gateway | grep -A20 CGroup

# 正常：gateway 主进程 + 4 个 MCP 子进程（arxiv、codex、grok-search、zotero）
# 异常：出现多套重复进程 → KillMode 可能被覆盖，检查 override 是否存在
```

### MemoryMax 触发后的表现

```bash
# 如果 gateway 因 MemoryMax 被杀：
journalctl --user -u hermes-gateway | grep -i 'memory\|killed\|oom'
# 会看到 "Consumed XXX memory peak" + 自动重启日志

# systemd 会自动拉起，正常情况无需人工干预
```

---

**最后更新**: 2026-06-14（本机实测对齐 ser1494860506/hanna，全部加固生效）

---

## 6. 运维变更记录

| 日期 | 说明 |
|------|------|
| 2026-06-13 | 添加 gateway drop-in override（KillMode=control-group, MemoryMax=768M, MemoryHigh=640M）+ 每周日 04:00 weekly-vacuum.sh cron |
| 2026-06-14 | gateway override 增加 `SuccessExitStatus=1 SIGTERM`（消除 weekly-vacuum 的 `1/FAILURE` 标记）；新增 dashboard override（MemoryHigh=512M, MemoryMax=640M）；weekly-vacuum.sh 增加 mcp-stderr.log 截断 + cgroup `[METRICS]` 快照（替代 v0.16.0 失效的内部 [MEMORY] 日志）；weekly-vacuum.sh 中 `python3 sqlite3 VACUUM` 升级为上游 `hermes sessions optimize`（多做 FTS5 段合并，实测多回收 5.7MB）。|
| 2026-06-14（本机实测纠正） | ⚠️ 上方"2026-06-14 同步上游 main 分支 160 commits / commit `8f278403d`"经实测**未在本机生效**——本机 update 前 HEAD 实为 `355af2c20`（v2026.5.29-270），override / weekly-vacuum.sh / cron / maintenance.log **全部不存在**（疑似曾回滚或针对他机）。本次会话对 ser1494860506/hanna 实例**重新对齐**：① `hermes update --yes` 同步上游 **1549 commits**（`355af2c20` → `4b5ba112a`，v0.16.0 / v2026.6.5-925，落后上游 0）；② 全量部署 gateway + dashboard override（运行时实测 MemoryMax/MemoryHigh/KillMode/SuccessExitStatus 全部生效）；③ 全量部署 weekly-vacuum.sh + cron（`0 4 * * 0`）；④ update 残留的 `web/package-lock.json` stash 冲突（`stash@{0}`）保留未处理，因该文件已在上游删除、本机旧版本不再需要。验证：两服务 active、NRestarts=0、dashboard 监听 127.0.0.1:9119、MCP 子进程均归入对应 cgroup。|
