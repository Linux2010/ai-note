# Discord 多 Bot 管理最佳实践 - 主 Agent 方案

> **文档类型**: 最佳实践  
> **适用场景**: OpenClaw 主 Agent 管理多个 Discord Bot，实现统一调度和权限控制  
> **最后更新**: 2026-03-19  
> **版本**: 1.0.0  
> **相关文档**: [Discord Bot 配置最佳实践](./discord-bot-configuration-best-practices.md)

---

## 📋 目录

1. [概述](#概述)
2. [架构设计](#架构设计)
3. [主 Agent 配置](#主-agent-配置)
4. [权限管理](#权限管理)
5. [多 Bot 协调](#多-bot-协调)
6. [安全考虑](#安全考虑)
7. [配置模板](#配置模板)

---

## 概述

本文档描述在 OpenClaw 中使用**主 Agent（main）统一管理多个 Discord Bot**的最佳实践方案。

### 核心原则

| 原则 | 说明 |
|------|------|
| **主 Agent 集中管理** | main Agent 拥有最高权限，协调其他 Bot |
| **最小权限原则** | 每个 Bot 只授予必要的权限 |
| **明确触发条件** | 只有特定用户@main 时才回复 |
| **Bot 隔离** | 其他 Bot 无法触发 main 的响应 |

---

## 架构设计

### 典型架构

```
┌─────────────────────────────────────────────────────────┐
│                    Discord 服务器                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  用户 (1482557771378593981)                             │
│       │                                                 │
│       ▼ @main                                           │
│  ┌─────────────┐                                        │
│  │ main Agent  │ ←── 主控制器（唯一响应@）              │
│  └──────┬──────┘                                        │
│         │ 调度                                          │
│    ┌────┴────┐                                          │
│    ▼         ▼                                          │
│ ┌──────┐ ┌──────┐                                       │
│ │Stock │ │Coding│  ←── 专业 Agent（不直接响应用户）     │
│ └──────┘ └──────┘                                       │
│                                                         │
│  其他 Bot（被忽略，无法触发 main）                       │
└─────────────────────────────────────────────────────────┘
```

### 角色定义

| 角色 | Bot Token | 权限 | 触发条件 |
|------|----------|------|----------|
| **main Agent** | 主 Token | 完全权限 | 仅允许用户@ |
| **stock Agent** | 独立 Token | 股票相关 | main 调度 |
| **coding Agent** | 独立 Token | 代码相关 | main 调度 |
| **其他 Bot** | N/A | 无 | 被忽略 |

---

## 主 Agent 配置

### 核心配置项

```json5
{
  channels: {
    discord: {
      // 必需：Bot Token
      token: "${DISCORD_BOT_TOKEN}",
      
      // 必需：代理配置（中国大陆）
      proxy: "http://127.0.0.1:7897",
      
      // 必需：只允许特定用户
      allowFrom: ["discord:1482557771378593981"],
      
      // 必需：开放群组模式
      groupPolicy: "open",
      
      // 必需：不允许其他 Bot 触发
      allowBots: false,
      
      // 必需：需要@main 才回复
      guilds: {
        "*": {
          requireMention: true
        }
      },
      
      // 可选：功能开关
      streaming: "off",
      
      // 可选：启用的操作
      actions: {
        reactions: true,
        sendMessage: true,
        deleteMessage: true,
        sticker: true,
        threads: true,
        pins: true
      }
    }
  },
  
  agents: {
    defaults: {
      model: "bailian/qwen3.5-plus",
      timeout: 60
    },
    list: [
      { id: "main", workspace: "~/.openclaw/agents/main/workspace" },
      { id: "stock", workspace: "~/.openclaw/agents/stock/workspace" },
      { id: "coding", workspace: "~/.openclaw/agents/coding/workspace" }
    ]
  }
}
```

---

## 权限管理

### 用户权限分级

| 用户类型 | allowFrom | requireMention | allowBots | 行为 |
|----------|-----------|----------------|-----------|------|
| **管理员** | ✅ | ✅ | ❌ | @main 触发 |
| **普通用户** | ❌ | ✅ | ❌ | 忽略 |
| **其他 Bot** | ❌ | ✅ | ❌ | 忽略 |

### 配置说明

```json5
{
  channels: {
    discord: {
      // 只允许管理员用户 ID
      allowFrom: ["discord:1482557771378593981"],
      
      // 群组中需要@main
      guilds: { "*": { requireMention: true } },
      
      // 不允许其他 Bot 触发
      allowBots: false
    }
  }
}
```

---

## 多 Bot 协调

### 主 Agent 调度模式

```
用户 @main → main Agent 接收
                ↓
         分析任务类型
                ↓
    ┌───────────┼───────────┐
    ↓           ↓           ↓
股票相关    代码相关    通用问题
    ↓           ↓           ↓
stock     coding       main 自己处理
Agent     Agent
```

### 调度实现

**方式 1: 子 Agent 调用**
```javascript
// main Agent 内部逻辑
if (topic === 'stock') {
  await sessions_spawn({ agentId: 'stock', task: userQuery });
} else if (topic === 'coding') {
  await sessions_spawn({ agentId: 'coding', task: userQuery });
}
```

**方式 2: 消息路由**
```json5
{
  bindings: [
    { 
      agentId: "main", 
      match: { channel: "discord", accountId: "default" } 
    },
    { 
      agentId: "stock", 
      match: { channel: "discord", accountId: "stock" } 
    }
  ]
}
```

---

## 安全考虑

### 1. Token 安全

```bash
# ✅ 正确：使用环境变量
DISCORD_BOT_TOKEN=xxx  # ~/.openclaw/.env

# ❌ 错误：硬编码在配置文件
```

### 2. 访问控制

```json5
{
  channels: {
    discord: {
      // 只允许特定用户
      allowFrom: ["discord:YOUR_USER_ID"],
      
      // 不允许 Bot 互触
      allowBots: false
    }
  }
}
```

### 3. 群组权限

```json5
{
  channels: {
    discord: {
      groupPolicy: "open",  // 开放但需要@
      guilds: {
        "*": {
          requireMention: true  // 必须@main
        }
      }
    }
  }
}
```

### 4. 代理配置

```json5
{
  channels: {
    discord: {
      // 中国大陆必需
      proxy: "http://127.0.0.1:7897"
    }
  }
}
```

---

## 配置模板

### 完整生产配置

```json5
// ~/.openclaw/openclaw.json
{
  channels: {
    discord: {
      token: "${DISCORD_BOT_TOKEN}",
      proxy: "http://127.0.0.1:7897",
      allowFrom: ["discord:1482557771378593981"],
      groupPolicy: "open",
      allowBots: false,
      streaming: "off",
      guilds: {
        "*": {
          requireMention: true,
          ignoreOtherMentions: true
        }
      },
      actions: {
        reactions: true,
        sendMessage: true,
        deleteMessage: true,
        sticker: true,
        threads: true,
        pins: true,
        permissions: false,
        moderation: false
      }
    }
  },
  
  agents: {
    defaults: {
      model: "bailian/qwen3.5-plus",
      timeout: 60,
      heartbeat: {
        every: "4h",
        directPolicy: "allow"
      }
    },
    list: [
      { 
        id: "main", 
        workspace: "~/.openclaw/agents/main/workspace" 
      },
      { 
        id: "stock", 
        workspace: "~/.openclaw/agents/stock/workspace" 
      },
      { 
        id: "coding", 
        workspace: "~/.openclaw/agents/coding/workspace" 
      }
    ]
  },
  
  bindings: [
    { 
      agentId: "main", 
      match: { channel: "discord", accountId: "default" } 
    }
  ]
}
```

### 环境变量配置

```bash
# ~/.openclaw/.env
DISCORD_BOT_TOKEN=MTQ4Mzk5ODMyMzYwMDc4NTU3OQ.Gw_jq5.xxx
DISCORD_PROXY=http://127.0.0.1:7897
```

---

## 故障排查

### 常见问题

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| Bot 不回复 | `allowBots: true` | 改为 `false` |
| 所有人都能触发 | `allowFrom` 未配置 | 添加用户 ID |
| 不需要@就回复 | `requireMention: false` | 改为 `true` |
| 连接超时 | 代理配置错误 | 检查代理地址 |
| 其他 Bot 能触发 | `allowBots: true` | 改为 `false` |

### 验证命令

```bash
# 查看当前配置
openclaw config get channels.discord

# 检查 Bot 状态
openclaw status --deep

# 查看日志
openclaw logs | grep discord
```

---

## 总结

### 核心配置清单

- [ ] `token`: Bot Token（环境变量）
- [ ] `proxy`: 代理配置（中国大陆必需）
- [ ] `allowFrom`: 只允许特定用户
- [ ] `allowBots`: `false`（不允许其他 Bot）
- [ ] `requireMention`: `true`（需要@main）
- [ ] `groupPolicy`: `open`（开放群组）

### 安全原则

```
✅ 最小权限：只允许必要用户
✅ 明确触发：必须@main
✅ Bot 隔离：不允许 Bot 互触
✅ Token 保护：使用环境变量
```

---

## 参考资源

- [Discord Bot 配置最佳实践](./discord-bot-configuration-best-practices.md)
- [OpenClaw 官方文档](https://docs.openclaw.ai/channels/discord)
- [多 Agent 路由](https://docs.openclaw.ai/concepts/multi-agent)

---

*本文档遵循 AI-Note 规范，欢迎提交 PR 改进。*
