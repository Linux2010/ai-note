# cc-connect Telegram Bot 无响应诊断与修复

```yaml
文档类型: 技术解决方案
适用场景: cc-connect 进程存活但收不到 Telegram 消息，需要排查 token、网络、代理问题
最后更新: 2026-07-15
```

## 问题描述

cc-connect 进程正在运行（`ps aux` 可见），但 Telegram bot 不响应消息。进程状态 SN（sleeping），CPU 0%，运行多天。

## 诊断步骤

### 1. 确认进程状态

```bash
ps aux | grep cc-connect | grep -v grep
```

进程存在但可能已经"僵死"——网络连接断开后无法恢复。

### 2. 验证 Bot Token 是否有效

```bash
# 从配置文件读取 token
cat ~/.cc-connect/config.toml | grep token

# 测试 token（需要代理的情况加 --proxy）
curl -s --proxy http://127.0.0.1:7897 \
  "https://api.telegram.org/bot<TOKEN>/getMe"
```

**返回值含义：**
- `{"ok":true,...}` → Token 有效
- `{"ok":false,"error_code":401}` → Token 已被撤销/失效
- `context deadline exceeded` → 网络/代理问题，连不上 Telegram API

### 3. 检查代理可用性

```bash
# 测试代理是否工作
curl -s --proxy http://127.0.0.1:7897 https://api.telegram.org/ -o /dev/null -w "%{http_code}"
```

### 4. 查看 cc-connect 日志

cc-connect 没有独立的日志文件，日志输出到 stderr。可通过 macOS unified log 查看：

```bash
log show --process cc-connect --last 10m --style compact
```

**常见错误模式：**
- `context deadline exceeded` → 网络/代理不通
- `Not Found` → Token 无效或 API URL 拼接错误
- `401 Unauthorized` → Token 被撤销

## 修复方案

### Token 失效

1. 去 Telegram 找 @BotFather
2. 发 `/mybots` → 选择目标 bot → API Token → Revoke and generate new
3. 更新配置：

```bash
# 编辑配置文件
vim ~/.cc-connect/config.toml
# 修改 token = "新token"

# 重启 cc-connect
kill $(pgrep -f cc-connect)
cc-connect -config ~/.cc-connect/config.toml &
```

### 网络问题

如果代理 `127.0.0.1:7897` 不可用：
1. 确认代理进程正在运行
2. 检查代理端口是否正确
3. 重启代理或更换代理

### 验证修复成功

```bash
# 方法 1：发送测试消息
curl -s --proxy http://127.0.0.1:7897 \
  "https://api.telegram.org/bot<TOKEN>/sendMessage?chat_id=<CHAT_ID>&text=test"

# 方法 2：检查 getUpdates（如果返回 409 Conflict 说明 cc-connect 正在 polling）
curl -s --proxy http://127.0.0.1:7897 \
  "https://api.telegram.org/bot<TOKEN>/getUpdates?limit=1"
# 409 Conflict = 有其他实例在 polling，说明 cc-connect 已连上 ✅
```

## 根因分析

cc-connect 无响应的典型根因链：

```
进程存活但无响应
├── Token 失效（被 BotFather 撤销 / Telegram 检测泄露）
│   └── 日志：401 Unauthorized 或 Not Found
├── 网络/代理问题
│   └── 日志：context deadline exceeded
└── 进程僵死（长时间运行后状态异常）
    └── CPU 0%，状态 SN，需要重启
```

**注意：** 旧进程可能缓存了旧 token 在内存中，即使配置文件已更新也不会重新加载。必须 kill 旧进程再启动。

## 配置参考

```toml
# ~/.cc-connect/config.toml
language = "zh"

[log]
level = "info"  # 排查问题时可改为 "debug"

[[projects]]
name = "hope-admin"

[projects.agent]
type = "claudecode"
work_dir = "/Users/hope/hope-project/hope-admin"

[[projects.platforms]]
type = "telegram"

[projects.platforms.options]
token = "<BOT_TOKEN>"
proxy = "http://127.0.0.1:7897"
```

## 改进建议

1. **添加 launchd plist**：进程挂掉后自动重启
2. **日志级别**：生产用 `info`，排查时切换 `debug`
3. **Token 监控**：定期 `getMe` 检查 token 是否有效，失效时告警
4. **健康检查**：监控 cc-connect 进程状态和网络连通性

## 相关文件

| 文件 | 用途 |
|------|------|
| `~/.cc-connect/config.toml` | 主配置文件 |
| `~/.cc-connect/sessions/*.json` | 会话状态 |
| `~/.cc-connect/run/api.sock` | API socket |
