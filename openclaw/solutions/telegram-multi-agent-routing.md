# OpenClaw Telegram 多频道配置问题解决方案

## 问题描述

在 OpenClaw 多 agent 配置中，Telegram 多 bot 设置存在以下问题：

- **症状**：所有私聊消息都被 Core Agent (main) 处理，而不是对应的特定 agent
- **原因**：bindings 配置只包含群组绑定规则，缺少私聊（direct message）绑定规则
- **影响**：无法通过私聊与 Stock Agent、MCS Agent、Coding Agent 等专用 agent 交互

## 根本原因分析

OpenClaw 的消息路由机制遵循以下优先级：
1. `peer` match (exact DM/group/channel id) - **最高优先级**
2. `accountId` match for a channel
3. channel-level match (`accountId: "*"`)
4. fallback to default agent (`main`) - **最低优先级**

当配置中只有群组绑定（包含 `peer` 字段）而没有私聊绑定时：
- 私聊消息不匹配任何 `peer` 规则
- 由于没有 `accountId` 级别的绑定规则
- 消息最终 fallback 到默认的 main agent

## 正确配置方案

### 错误配置示例
```json
"bindings": [
  {
    "agentId": "stock",
    "match": {
      "channel": "telegram", 
      "accountId": "stock",
      "peer": {
        "kind": "group",
        "id": "-1003661163902"
      }
    }
  }
  // 缺少私聊绑定规则！
]
```

### 正确配置方案
每个 agent 需要 **两个绑定规则**：

1. **群组绑定**：用于群组中的 @ 提及
2. **账户绑定**：用于私聊和其他未指定 peer 的消息

```json
"bindings": [
  // Stock Agent - 群组绑定
  {
    "agentId": "stock",
    "match": {
      "channel": "telegram",
      "accountId": "stock", 
      "peer": {
        "kind": "group",
        "id": "-1003661163902"
      }
    }
  },
  // Stock Agent - 私聊绑定  
  {
    "agentId": "stock",
    "match": {
      "channel": "telegram",
      "accountId": "stock"
    }
  },
  
  // MCS Agent - 群组绑定
  {
    "agentId": "mcs",
    "match": {
      "channel": "telegram",
      "accountId": "mcs",
      "peer": {
        "kind": "group", 
        "id": "-1003661163902"
      }
    }
  },
  // MCS Agent - 私聊绑定
  {
    "agentId": "mcs",
    "match": {
      "channel": "telegram", 
      "accountId": "mcs"
    }
  },
  
  // Coding Agent - 群组绑定
  {
    "agentId": "coding",
    "match": {
      "channel": "telegram",
      "accountId": "coding",
      "peer": {
        "kind": "group",
        "id": "-1003661163902" 
      }
    }
  },
  // Coding Agent - 私聊绑定
  {
    "agentId": "coding", 
    "match": {
      "channel": "telegram",
      "accountId": "coding"
    }
  },
  
  // Core Agent - 群组绑定
  {
    "agentId": "main",
    "match": {
      "channel": "telegram",
      "accountId": "core",
      "peer": {
        "kind": "group",
        "id": "-1003661163902"
      }
    }
  },
  // Core Agent - 私聊绑定  
  {
    "agentId": "main",
    "match": {
      "channel": "telegram",
      "accountId": "core"
    }
  }
]
```

## 验证配置

使用以下命令验证配置是否正确：

```bash
# 查看 agent 绑定规则
openclaw agents list --bindings

# 重启 Gateway 应用配置
openclaw gateway restart

# 检查状态
openclaw status
```

正确的输出应该显示每个 agent 都有 **2 个 routing rules**：

```
Agents:
- main (default)
  Routing rules: 2
  Routing rules:
    - telegram accountId=core peer=group:-1003661163902
    - telegram accountId=core
- stock
  Routing rules: 2  
  Routing rules:
    - telegram accountId=stock peer=group:-1003661163902
    - telegram accountId=stock
- mcs
  Routing rules: 2
  Routing rules:
    - telegram accountId=mcs peer=group:-1003661163902
    - telegram accountId=mcs
- coding
  Routing rules: 2
  Routing rules:
    - telegram accountId=coding peer=group:-1003661163902
    - telegram accountId=coding
```

## 测试方法

1. **私聊测试**：
   - 私聊 `@one_asset_bot` → 应该收到 Stock Agent 回复
   - 私聊 `@one_mcs_bot` → 应该收到 MCS Agent 回复  
   - 私聊 `@coding_bot` → 应该收到 Coding Agent 回复
   - 私聊 `@meme_hope_bot` → 应该收到 Core Agent 回复

2. **群组测试**：
   - 在群组 `-1003661163902` 中 @one_asset_bot → Stock Agent 回复
   - 在群组 `-1003661163902` 中 @one_mcs_bot → MCS Agent 回复
   - 在群组 `-1003661163902` 中 @coding_bot → Coding Agent 回复
   - 在群组 `-1003661163902` 中 @meme_hope_bot → Core Agent 回复

## 常见陷阱

### 1. 只配置群组绑定
❌ 错误：只添加了包含 `peer` 的绑定规则
✅ 正确：必须同时添加群组和私聊绑定规则

### 2. accountId 不匹配  
❌ 错误：binding 中的 `accountId` 与 channels 配置中的不一致
✅ 正确：确保 `bindings[].match.accountId` 与 `channels.telegram.accounts` 中的 key 完全匹配

### 3. 忘记重启 Gateway
❌ 错误：修改配置后没有重启服务
✅ 正确：使用 `openclaw gateway restart` 应用新配置

## 参考文档

- [OpenClaw Multi-Agent 官方文档](https://docs.openclaw.ai/concepts/multi-agent)
- [Telegram Bots per Agent 示例](https://docs.openclaw.ai/concepts/multi-agent#telegram-bots-per-agent)

## 版本信息

- **OpenClaw 版本**: 2026.2.26
- **解决方案验证日期**: 2026-02-28
- **适用场景**: Telegram 多 bot + 多 agent 配置

---
*本文档由 AI 自动生成并验证，适用于 OpenClaw 多 agent Telegram 配置场景*