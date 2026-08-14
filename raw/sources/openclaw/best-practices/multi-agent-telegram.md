# OpenClaw 多代理 Telegram 交互最佳实践

## 背景与限制

在 OpenClaw 多代理架构中，通过 Telegram 实现多代理协作存在以下技术限制：

### Telegram 平台限制
- **机器人隔离**: Telegram 机器人无法读取其他机器人的消息（包括 @ 提及）
- **Privacy Mode**: 默认启用隐私模式，限制机器人接收群组消息
- **消息路由**: 只有被明确 @ 的机器人才会收到对应消息

### OpenClaw 当前版本限制
- **Issue #408**: Bots can't read other bot's messages on Telegram
- **Issue #29173**: 多机器人 mention 冲突问题（substring 匹配错误）
- **路由机制**: 只处理 inbound 消息，outbound 消息不会重新路由

## 推荐方案：直接 @ 交互模式

基于当前技术限制，**直接 @ 交互是最稳定可靠的方案**。

### 架构设计
```
Telegram Group "one" (-1003661163902)
├── @core_yourname_bot → Core Agent (主协调者)
├── @stock_yourname_bot → Stock Agent (投资专家)  
├── @mcs_yourname_bot → MCS Agent (CS技术专家)
└── @code_yourname_bot → Code Agent (开源代码专家)
```

### 配置要点

#### 1. 多账号配置
```json5
{
  "channels": {
    "telegram": {
      "accounts": {
        "core": { "botToken": "CORE_TOKEN" },
        "stock": { "botToken": "STOCK_TOKEN" },
        "mcs": { "botToken": "MCS_TOKEN" },
        "code": { "botToken": "CODE_TOKEN" }
      }
    }
  }
}
```

#### 2. 群组设置
```json5
{
  "channels": {
    "telegram": {
      "groups": {
        "-1003661163902": {
          "requireMention": true,
          "groupPolicy": "open"
        }
      }
    }
  }
}
```

#### 3. 绑定规则
```json5
{
  "bindings": [
    { "agentId": "stock", "match": { "channel": "telegram", "accountId": "stock" } },
    { "agentId": "mcs", "match": { "channel": "telegram", "accountId": "mcs" } },
    { "agentId": "code", "match": { "channel": "telegram", "accountId": "code" } },
    { "agentId": "main", "match": { "channel": "telegram", "accountId": "core" } }
  ]
}
```

### 使用规范

#### 用户交互方式
- **@core_yourname_bot**: 综合协调、系统管理、跨领域问题
- **@stock_yourname_bot**: 投资组合分析、市场监控、交易合规
- **@mcs_yourname_bot**: CS认证指导、技术面试准备、职业发展
- **@code_yourname_bot**: 开源项目贡献、issue解决、PR提交

#### 行为准则
- **无@消息**: 所有代理保持静默，避免干扰
- **精准@**: 明确指定需要的专业代理
- **职责分离**: 各代理专注于自己的专业领域

## 配置实施步骤

### 1. 创建 Telegram 机器人
- 通过 @BotFather 为每个代理创建独立的 bot
- 记录每个 bot 的 token

### 2. 配置 Privacy Mode
- 对每个机器人执行 `/setprivacy` → **Disable**
- **重要**: 每个机器人都需要单独禁用 Privacy Mode

### 3. 添加机器人到群组
- 将所有四个机器人添加到目标群组
- 确保群组 ID 正确（格式：-100xxxxxxxxxx）

### 4. 应用 OpenClaw 配置
- 更新 `openclaw.json` 包含多账号和绑定配置
- 重启 OpenClaw Gateway 应用配置

### 5. 验证功能
- 测试每个 @ 提及是否正确路由到对应代理
- 确认无@消息时所有代理保持静默

## 未来演进方向

### 等待官方支持
- **Telegram API 改进**: 支持机器人间消息读取
- **OpenClaw 修复**: 解决 Issue #408 和 #29173
- **高级路由**: 支持代理间内部通讯 + 群组回复

### 临时增强方案
如果需要更复杂的协作，可以考虑：
- **Core 代理协调模式**: 只保留 Core 机器人在群组中，通过指令前缀调用其他代理
- **多群组分离**: 为每个代理创建专用群组

## 故障排除

### 常见问题
1. **机器人不响应 @ 提及**
   - 检查 Privacy Mode 是否已禁用
   - 确认机器人已重新添加到群组
   - 验证 binding 配置是否正确

2. **多个机器人同时响应**
   - 检查 mention 检测是否存在 substring 冲突
   - 确保机器人用户名没有包含关系

3. **群组消息完全不接收**
   - 验证群组 ID 格式是否正确
   - 检查 `requireMention` 配置

### 调试命令
```bash
# 查看代理绑定状态
openclaw agents list --bindings

# 检查通道状态  
openclaw channels status

# 查看日志
openclaw logs --follow
```

---
**文档类型**: 最佳实践  
**适用场景**: OpenClaw 多代理 Telegram 集成  
**最后更新**: 2026-02-28