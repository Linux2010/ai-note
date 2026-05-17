# OpenClaw Telegram 长轮询断连 — Event-Loop 饥饿修复

**创建时间：** 2026-05-17  
**问题类型：** Gateway 进程 / 事件循环饥饿 / 长轮询断连  
**影响范围：** 本地部署 OpenClaw + Telegram 集成（非 Docker）  
**适用版本：** OpenClaw v2026.5.5+

---

## 🔍 问题现象

### 症状
- Telegram Bot 主动发送消息到群组 **成功**（sendMessage API 通）
- Telegram Bot **收不到群组消息**（getUpdates 长轮询失败）
- 日志中出现大量 `fetch-timeout`，目标为 `https://api.telegram.org/bot.../getMe`
- Health Monitor 反复重启 Telegram 插件：`reason: disconnected`
- 最终陷入恶性循环：重启插件 → 断连 → 再重启

### 用户反馈
> "telegram 群组为什么无法通讯了"

---

## 📊 诊断过程

### 1. 验证 Bot Token 有效性 ✅
```bash
curl -s --connect-timeout 10 -x http://127.0.0.1:7897 \
  "https://api.telegram.org/bot<TOKEN>/getMe"

# 结果：Bot 在线，配置正常
{"ok":true,"result":{"id":8524712381,"is_bot":true,"username":"meme_hope_bot",...}}
```

### 2. 验证代理连通性 ✅
```bash
curl -s --connect-timeout 5 -x http://127.0.0.1:7897 https://api.telegram.org/
# HTTP 302 — 代理正常工作
```

### 3. 检查 Gateway 日志 ❌

**gateway.log** — Telegram 插件反复重启：
```
[telegram] [core] starting provider (@meme_hope_bot)
[health-monitor] [telegram:core] health-monitor: restarting (reason: disconnected)
[telegram] [core] starting provider (@meme_hope_bot)
[health-monitor] [telegram:core] health-monitor: restarting (reason: disconnected)
```

**gateway.err.log** — 大量 fetch-timeout：
```
[fetch-timeout] fetch timeout after 10000ms (elapsed 10001ms)
  operation=fetchWithTimeout url=https://api.telegram.org/bot.../getMe
[fetch-timeout] fetch timeout after 10000ms (elapsed 29872ms)
  timer delayed 19872ms, likely event-loop starvation
```

### 4. 检查 Gateway 进程
```bash
ps aux | grep -i openclaw | grep -v grep
# PID 6176, 启动时间: Fri May 15 07:24:24 2026
# 进程已运行超过 48 小时
```

### 5. 测试主动发送消息 ✅
```bash
# 通过 message 工具发送到群组 -1003661163902
# 结果：成功，messageId=365
```

---

## 🎯 根因分析

### 核心原因：Event-Loop 饥饿

| 现象 | 说明 |
|------|------|
| sendMessage 成功 | 单次 HTTP 请求，不依赖长轮询 |
| getUpdates 超时 | 长轮询连接持续 10s+ 无响应 |
| `timer delayed 19872ms` | 事件循环阻塞 ~20s 无法执行定时器回调 |
| `event-loop starvation` | 日志明确标注了饥饿原因 |

### 饥饿的触发链路

1. **Cron 定时任务超时** — 某个 cron 任务的模型调用超时（150s）
2. **模型调用阻塞** — 百炼模型 API 响应慢，主线程长时间等待
3. **Event-Loop 饥饿** — Node.js 事件循环被阻塞，无法处理网络 I/O
4. **长轮询断连** — Telegram getUpdates 超时，Telegram 服务器认为客户端离线
5. **健康监控重启** — health-monitor 检测到断连，重启 Telegram 插件
6. **恶性循环** — 重启后仍然饥饿 → 再次断连 → 再次重启

### 为什么 SIGUSR1 热重启也失效？

`SIGUSR1` 需要事件循环处理信号回调，但当 event-loop 完全饥饿时：
```
[gateway-tool] gateway tool: restart requested
# 之后没有重启日志 — 信号被排队但无法执行
```
进程状态：PID 不变，启动时间不变（仍然是 5 月 15 日的进程）。

---

## ✅ 解决方案

### 方案一：硬重启 Gateway（推荐，立即恢复）

```bash
# 查找 Gateway 进程
ps aux | grep "openclaw.*gateway" | grep -v grep

# 发送 SIGTERM 强制终止
kill <PID>

# 等待服务管理自动重启（或用 systemd/launchd 重启）
# OpenClaw 通常有守护进程自动拉起
```

### 方案二：通过 Gateway 工具重启
```bash
# 如果 SIGUSR1 能响应
openclaw gateway restart
```

### 方案三：launchd 重启（macOS）
```bash
# 如果通过 launchd 管理
launchctl stop com.openclaw.gateway
launchctl start com.openclaw.gateway
```

---

## 📋 预防措施

### 1. 定期重启 Gateway
```bash
# 建议每天或每 24-48 小时重启一次
# 可通过 cron 任务实现：
openclaw cron add --name "gateway-restart" \
  --cron "0 4 * * *" --tz "Asia/Shanghai" \
  --session main \
  --system-event "重启 Gateway 释放资源，防止 event-loop 饥饿"
```

### 2. 优化 Cron 任务超时配置
检查长时间运行的 cron 任务，设置合理的 `timeoutSeconds`：
```json
// cron jobs.json 中
{
  "payload": {
    "kind": "agentTurn",
    "timeoutSeconds": 120  // 不要设太大
  }
}
```

### 3. 监控日志
定期检查 `gateway.err.log` 中的关键词：
```bash
# 搜索 event-loop starvation 警告
grep "event-loop starvation" ~/.openclaw/logs/gateway.err.log

# 搜索 fetch-timeout 频率
grep -c "fetch-timeout" ~/.openclaw/logs/gateway.err.log
```

### 4. 设置告警
如果 fetch-timeout 在短时间内频繁出现，说明 event-loop 开始饥饿，
应在健康监控介入前主动重启。

---

## 🛠️ 诊断命令清单

### 检查 Telegram 状态
```bash
# Bot 在线状态
curl -s -x http://127.0.0.1:7897 \
  "https://api.telegram.org/bot<TOKEN>/getMe"

# 主动发送测试消息
curl -s -x http://127.0.0.1:7897 \
  "https://api.telegram.org/bot<TOKEN>/sendMessage" \
  -d "chat_id=<GROUP_ID>" -d "text=测试消息"
```

### 检查 Gateway 健康
```bash
# 进程运行时间（越短越好，说明最近重启过）
ps -o pid,lstart,command -p $(pgrep -f "openclaw.*gateway")

# 最近 50 行日志中的 Telegram 相关日志
tail -50 ~/.openclaw/logs/gateway.log | grep -i telegram

# 错误日志中的超时统计
grep -c "fetch-timeout" ~/.openclaw/logs/gateway.err.log
```

### 检查代理
```bash
# 代理端口是否在监听
lsof -i :7897

# 代理连通性
curl -s --connect-timeout 5 -x http://127.0.0.1:7897 https://api.telegram.org/
```

---

## 🔗 相关文档

- [Docker 容器内 Telegram Bot 连接问题排查](./docker-telegram-proxy-connection-issue.md)
- [Telegram 多频道路由配置](./telegram-multi-agent-routing.md)
- [代理配置综合方案](./proxy-solutions.md)

---

## 📝 更新记录

| 日期 | 内容 | 作者 |
|------|------|------|
| 2026-05-17 | 初始版本 - 记录 event-loop 饥饿导致 Telegram 长轮询断连问题排查与修复 | Leader 🦞 |

---

**标签：** #OpenClaw #Telegram #EventLoop #长轮询 #故障排查 #NodeJS  
**适用版本：** OpenClaw v2026.5.5+, macOS / Linux
