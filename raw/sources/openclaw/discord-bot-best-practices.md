# Discord Bot 配置最佳实践 - OpenClaw Trump 实例

**创建时间：** 2026-03-18  
**作者：** Hope (OpenClaw Agent)  
**状态：** ✅ 已验证

---

## 📋 概述

本文档记录在 OpenClaw 多实例环境下配置 Discord Bot 的完整流程和最佳实践，基于 Trump 实例的成功配置经验。

---

## 🎯 核心场景

### 场景 1：多实例共用同一个 Bot

**适用情况：**
- 测试/学习环境
- 单个 Bot 服务多个实例
- 不介意回复冲突

**配置方式：**
```json
// 所有实例使用相同的 Token
{
  "channels": {
    "discord": {
      "enabled": true,
      "token": "你的_BOT_TOKEN",
      "proxy": "http://127.0.0.1:7897"
    }
  }
}
```

**⚠️ 注意：** 多个实例会同时收到消息并可能同时回复

---

### 场景 2：每个实例独立 Bot

**适用情况：**
- 生产环境
- 需要独立的 Bot 身份
- 避免回复冲突

**配置方式：**
- 每个实例注册独立的 Discord Bot
- 使用不同的 Token
- 可以添加到同一个服务器

---

## 📖 完整配置流程

### 第一步：创建 Discord Bot

1. 访问 [Discord Developer Portal](https://discord.com/developers/applications)
2. 点击 **New Application**
3. 输入应用名称（如 "Trump Bot"）
4. 点击左侧 **Bot** → **Add Bot**
5. 设置 Bot 用户名和头像

### 第二步：启用 Privileged Intents

在 **Bot** 页面，向下滚动到 **Privileged Gateway Intents**：

- ✅ **Message Content Intent**（必需）
- ✅ **Server Members Intent**（推荐）
- ⭕ Presence Intent（可选）

### 第三步：获取配置信息

需要收集三个关键信息：

| 信息 | 获取方式 | 示例 |
|------|----------|------|
| **Bot Token** | Bot 页面 → Reset Token | 保密，不公开 |
| **服务器 ID** | 右键服务器图标 → 复制服务器 ID | `1483648697886707933` |
| **用户 ID** | 右键自己头像 → 复制用户 ID | `1482557771378593981` |

**⚠️ 重要提示：**
- 开启 **开发者模式** 才能复制 ID（设置 → 高级 → 开发者模式）
- Token 只显示一次，务必保存好
- **不要** 把 Telegram ID 当成 Discord ID！

### 第四步：配置 OpenClaw

编辑实例配置文件（如 `~/.openclaw-trump/openclaw.json`）：

```json
{
  "channels": {
    "discord": {
      "enabled": true,
      "token": "你的_BOT_TOKEN",
      "proxy": "http://127.0.0.1:7897",
      "groupPolicy": "allowlist",
      "streaming": "off",
      "maxLinesPerMessage": 20,
      "dmPolicy": "allowlist",
      "allowFrom": ["你的_Discord_用户_ID"],
      "guilds": {
        "你的_服务器_ID": {
          "requireMention": false,
          "users": ["你的_Discord_用户_ID"]
        }
      }
    }
  }
}
```

**关键字段说明：**

| 字段 | 说明 | 推荐值 |
|------|------|--------|
| `proxy` | 网络代理（国内必需） | `http://127.0.0.1:7897` |
| `groupPolicy` | 服务器访问策略 | `allowlist` |
| `dmPolicy` | 私信访问策略 | `allowlist` |
| `allowFrom` | 允许的用户 ID 列表 | `["你的_Discord_用户_ID"]` |
| `requireMention` | 是否需要 @Bot | `false`（私人服务器） |

### 第五步：重启 Gateway

```bash
# Trump 实例
cd /Users/kylin/.openclaw-trump
OPENCLAW_DIR=/Users/kylin/.openclaw-trump npx openclaw gateway restart

# 主实例
npx openclaw gateway restart
```

### 第六步：验证配置

```bash
# 检查状态
npx openclaw channels status 2>&1 | grep Discord

# 预期输出：
# - Discord default: enabled, configured, running, connected, token:config
```

---

## 🔧 常见问题

### 问题 1：Bot 离线/无法连接

**症状：** `Discord default: ... disconnected`

**原因：** 网络问题，国内需要代理

**解决：**
```json
{
  "discord": {
    "proxy": "http://127.0.0.1:7897"
  }
}
```

### 问题 2：能收消息但不能回复

**症状：** Web 端有回复，Discord 收不到

**日志：** `discord final reply failed: TypeError: fetch failed`

**原因：**
1. 代理配置错误
2. 用户 ID 配置错误（用了 Telegram ID）

**解决：**
- 确保 `pro