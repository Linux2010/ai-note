# Discord Bot 配置最佳实践

> **文档类型**: 最佳实践  
> **适用场景**: OpenClaw 集成 Discord 作为 AI 代理通信渠道  
> **最后更新**: 2026-03-18  
> **版本**: 1.0.0

---

## 📋 目录

1. [概述](#概述)
2. [前置准备](#前置准备)
3. [Bot 创建与配置](#bot-创建与配置)
4. [OpenClaw 配置](#openclaw-配置)
5. [权限管理](#权限管理)
6. [安全最佳实践](#安全最佳实践)
7. [故障排查](#故障排查)
8. [进阶配置](#进阶配置)

---

## 概述

本文档提供在 OpenClaw 中配置 Discord Bot 的完整最佳实践，涵盖从 Bot 创建到生产部署的全流程。

### 为什么选择 Discord

| 优势 | 说明 |
|------|------|
| **多频道支持** | 支持服务器、私聊、线程等多种场景 |
| **丰富的 API** | 完整的消息、角色、权限管理 |
| **开发者友好** | 完善的文档和活跃的社区 |
| **免费额度充足** | 适合个人和中小型项目 |

---

## 前置准备

### 必需项

- ✅ Discord 开发者账号
- ✅ Node.js 22+ (OpenClaw 运行环境)
- ✅ OpenClaw 已安装并运行
- ✅ 稳定的网络连接（Discord API 访问）

### 推荐项

- ✅ 独立的 Discord 服务器（用于测试）
- ✅ 环境变量管理工具（如 dotenv）
- ✅ Git 版本控制

---

## Bot 创建与配置

### 步骤 1: 创建 Discord 应用

1. 访问 [Discord Developer Portal](https://discord.com/developers/applications)
2. 点击 **"New Application"**
3. 输入应用名称（如：OpenClaw-Bot）
4. 点击 **"Create"**

### 步骤 2: 创建 Bot 用户

1. 在应用页面，点击左侧 **"Bot"**
2. 点击 **"Add Bot"** → **"Yes, do it!"**
3. 记录 **Bot Token**（⚠️ 只显示一次！）

```
⚠️ 安全警告：
- Bot Token 相当于密码，绝不提交到 Git
- 不要在任何公开场合分享 Token
- 如 Token 泄露，立即点击 "Reset Token"
```

### 步骤 3: 配置 Bot 权限

推荐权限配置：

```
✅ Send Messages          - 发送消息
✅ Read Message History   - 读取历史消息
✅ Embed Links            - 发送富文本
✅ Attach Files           - 发送文件
✅ Use Slash Commands     - 使用斜杠命令
✅ Manage Threads         - 管理线程（如需要）
```

**最小权限原则**: 只授予必要的权限。

### 步骤 4: 邀请 Bot 到服务器

1. 左侧菜单 **"OAuth2"** → **"URL Generator"**
2. 选择 scopes:
   - ✅ `bot`
   - ✅ `applications.commands`（斜杠命令）
3. 选择 Bot Permissions（见上一步）
4. 复制生成的 URL 到浏览器
5. 选择服务器，点击 **"Authorize"**

**邀请 URL 格式**:
```
https://discord.com/api/oauth2/authorize?client_id=YOUR_CLIENT_ID&permissions=PERMISSIONS&scope=bot%20applications.commands
```

---

## OpenClaw 配置

### 方式 1: 使用配置命令（推荐）

```bash
# 设置 Discord Bot Token
openclaw config set channels.discord.botToken "YOUR_BOT_TOKEN"

# 设置允许的用户 ID（可选，限制访问）
openclaw config set channels.discord.allowFrom ["123456789012345678"]

# 设置监听的消息前缀（如需要）
openclaw config set channels.discord.prefix "!"
```

### 方式 2: 直接编辑配置文件

编辑 `~/.openclaw/openclaw.json`:

```json5
{
  channels: {
    discord: {
      botToken: "YOUR_BOT_TOKEN_HERE",
      allowFrom: ["123456789012345678"],  // 可选：限制用户
      prefix: "!",                        // 可选：消息前缀
      enableThreads: true,                // 启用线程支持
      enableReactions: true               // 启用表情反应
    }
  },
  agents: {
    defaults: {
      model: "bailian/qwen3.5-plus"
    }
  }
}
```

### 方式 3: 使用环境变量（生产环境推荐）

创建 `~/.openclaw/.env`:

```bash
DISCORD_BOT_TOKEN=YOUR_BOT_TOKEN_HERE
```

OpenClaw 会自动读取环境变量。

---

## 权限管理

### 用户权限分级

| 角色 | 权限 | 配置方式 |
|------|------|----------|
| **管理员** | 所有命令 + 配置修改 | `allowFrom` + 管理员角色 |
| **普通用户** | 基础对话命令 | `allowFrom` 列表 |
| **受限用户** | 只读或有限命令 | 自定义权限组 |

### Discord 角色映射

在 `openclaw.json` 中配置角色权限：

```json5
{
  channels: {
    discord: {
      rolePermissions: {
        "Admin": ["all"],
        "Moderator": ["chat", "manage-threads"],
        "User": ["chat"]
      }
    }
  }
}
```

### 频道访问控制

```json5
{
  channels: {
    discord: {
      channelRestrictions: {
        "123456789012345678": {  // 频道 ID
          allow: true,
          rateLimit: 10  // 每分钟消息数限制
        }
      }
    }
  }
}
```

---

## 安全最佳实践

### 1. Token 安全

```bash
# ✅ 正确：使用环境变量
DISCORD_BOT_TOKEN=xxx

# ❌ 错误：硬编码在配置文件中
{
  "discord": {
    "botToken": "xxx"  // 不要这样做！
  }
}
```

### 2. 访问控制

```json5
// 限制只有特定用户可以访问
{
  channels: {
    discord: {
      allowFrom: ["user_id_1", "user_id_2"]
    }
  }
}
```

### 3. 速率限制

防止滥用和 API 限流：

```json5
{
  channels: {
    discord: {
      rateLimit: {
        perUser: 10,      // 每用户每分钟消息数
        perChannel: 100,  // 每频道每分钟消息数
        burst: 5          // 突发消息数
      }
    }
  }
}
```

### 4. 日志记录

启用详细日志用于调试：

```bash
openclaw config set logging.channels.discord "debug"
```

### 5. 定期轮换 Token

建议每 3-6 个月轮换一次 Bot Token：

```bash
# 在 Discord Developer Portal 重置 Token
# 然后更新配置
openclaw config set channels.discord.botToken "NEW_TOKEN"
```

---

## 故障排查

### 常见问题

#### 1. Bot 无法连接

**症状**: Bot 离线，无法响应消息

**检查清单**:
```bash
# 检查 Token 是否正确
openclaw config get channels.discord.botToken

# 检查网络连接
ping discord.com

# 查看日志
openclaw logs | grep discord
```

**解决方案**:
- 验证 Token 是否正确
- 检查 Bot 是否在服务器中在线
- 确认网络可以访问 Discord API

#### 2. Bot 无法发送消息

**症状**: 可以接收消息但无法回复

**可能原因**:
- 缺少发送消息权限
- 频道权限配置错误
- Bot 角色权限不足

**解决方案**:
1. 在 Discord 服务器设置中检查 Bot 角色权限
2. 确认频道允许 Bot 发送消息
3. 检查 OpenClaw 日志中的权限错误

#### 3. 消息响应延迟

**症状**: Bot 回复很慢或超时

**可能原因**:
- 模型 API 响应慢
- 网络延迟
- 服务器负载高

**解决方案**:
```bash
# 检查模型状态
openclaw status

# 调整超时配置
openclaw config set agents.defaults.timeout 60
```

#### 4. 斜杠命令不响应

**症状**: `/command` 无响应

**解决方案**:
1. 确认添加了 `applications.commands` scope
2. 重新邀请 Bot 到服务器
3. 等待 Discord 缓存刷新（可能需要几分钟）

---

## 进阶配置

### 多服务器支持

```json5
{
  channels: {
    discord: {
      servers: {
        "server_id_1": {
          prefix: "!",
          enableCommands: true
        },
        "server_id_2": {
          prefix: "?",
          enableCommands: false
        }
      }
    }
  }
}
```

### 自定义命令

```json5
{
  channels: {
    discord: {
      customCommands: {
        "/help": {
          response: "可用命令：/chat, /status, /help",
          ephemeral: true
        },
        "/status": {
          response: "Bot 运行正常 ✅",
          ephemeral: true
        }
      }
    }
  }
}
```

### 事件监听器

```json5
{
  channels: {
    discord: {
      eventListeners: {
        "messageReactionAdd": true,
        "threadCreate": true,
        "memberJoin": false
      }
    }
  }
}
```

### Webhook 集成

```json5
{
  channels: {
    discord: {
      webhooks: {
        "notifications": {
          url: "https://discord.com/api/webhooks/xxx",
          events: ["error", "warning"]
        }
      }
    }
  }
}
```

---

## 性能优化

### 1. 消息缓存

```json5
{
  channels: {
    discord: {
      cache: {
        enabled: true,
        ttl: 300,  // 5 分钟
        maxSize: 1000
      }
    }
  }
}
```

### 2. 连接池

```json5
{
  channels: {
    discord: {
      connection: {
        maxReconnectAttempts: 5,
        reconnectDelay: 1000,
        heartbeatInterval: 30000
      }
    }
  }
}
```

### 3. 消息批处理

```json5
{
  channels: {
    discord: {
      batching: {
        enabled: true,
        maxSize: 10,
        maxDelay: 100
      }
    }
  }
}
```

---

## 监控与告警

### 健康检查端点

```bash
# 启用健康检查
openclaw config set monitoring.discord.enabled true
```

### 关键指标

| 指标 | 阈值 | 告警级别 |
|------|------|----------|
| 消息延迟 | > 5s | Warning |
| 错误率 | > 5% | Critical |
| 连接断开 | > 3 次/小时 | Warning |
| API 限流 | 接近限制 | Warning |

### 告警配置

```json5
{
  monitoring: {
    discord: {
      alerts: {
        webhook: "https://your-webhook-url/alerts",
        thresholds: {
          messageDelay: 5000,
          errorRate: 0.05
        }
      }
    }
  }
}
```

---

## 总结

### 快速检查清单

- [ ] Bot Token 已安全存储（环境变量）
- [ ] Bot 权限已最小化配置
- [ ] 访问控制已启用（allowFrom）
- [ ] 速率限制已配置
- [ ] 日志记录已启用
- [ ] 监控告警已设置

### 推荐配置模板

```json5
// ~/.openclaw/openclaw.json
{
  channels: {
    discord: {
      botToken: "${DISCORD_BOT_TOKEN}",  // 从环境变量读取
      allowFrom: ["admin_user_id"],
      prefix: "!",
      enableThreads: true,
      enableReactions: true,
      rateLimit: {
        perUser: 10,
        perChannel: 100
      },
      logging: "info"
    }
  },
  agents: {
    defaults: {
      model: "bailian/qwen3.5-plus",
      timeout: 60
    }
  }
}
```

---

## 参考资源

- [Discord Developer Documentation](https://discord.com/developers/docs)
- [OpenClaw Discord Channel Docs](https://docs.openclaw.ai/channels/discord)
- [Discord API 限流说明](https://discord.com/developers/docs/topics/rate-limits)

---

*本文档遵循 AI-Note 规范，欢迎提交 PR 改进。*
