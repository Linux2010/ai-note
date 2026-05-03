# Claude Code 配置百炼 Coding Plan 最佳实践

> **日期**: 2026-05-03  
> **版本**: 1.0  
> **作者**: Linux2010  
> **标签**: `claude-code` `百炼` `coding-plan` `配置`

---

## 一、概述

本文档记录如何在 Anthropic Claude Code 中配置阿里云百炼 Coding Plan 模型，实现低成本、高质量的 AI 编码体验。

### 核心优势

| 优势 | 说明 |
|------|------|
| **零成本** | Coding Plan 模型免费使用（无 token 费用） |
| **多模型** | 支持 glm-5、qwen3.6-plus 等多个模型 |
| **OpenAI 兼容** | 通过 Anthropic 兼容接口调用 |
| **本地运行** | 代码不离开本地，保护隐私 |

---

## 二、环境准备

### 2.1 获取百炼 API Key

1. 访问 [百炼控制台](https://bailian.console.aliyun.com/)
2. 登录阿里云账号
3. 进入 **API-KEY 管理**
4. 创建或复制现有 API Key（格式：`sk-sp-xxxxxxxx`）

### 2.2 安装 Claude Code

```bash
# 方式 1: npm 全局安装
npm install -g @anthropic-ai/claude-code

# 方式 2: VS Code 扩展
# 安装 "Claude Code" 扩展（anthropic.claude-code-*）
```

---

## 三、配置文件

### 3.1 配置文件位置

Claude Code 的配置文件位于：

```
~/.claude/settings.json
```

### 3.2 完整配置示例

```json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "sk-sp-你的API_KEY",
    "ANTHROPIC_BASE_URL": "https://coding.dashscope.aliyuncs.com/apps/anthropic",
    "ANTHROPIC_MODEL": "glm-5",
    "CLAUDE_FALLBACK_MODEL": "qwen3.6-plus"
  },
  "permissionMode": "bypass",
  "permissions": {
    "allow": [
      "Bash(*)",
      "Read",
      "Edit",
      "Write",
      "WebFetch",
      "WebSearch",
      "Glob",
      "Grep",
      "TodoWrite",
      "Agent(*)"
    ]
  },
  "enabledPlugins": {
    "telegram@claude-plugins-official": true
  },
  "effortLevel": "xhigh",
  "model": "glm-5"
}
```

### 3.3 关键字段说明

| 字段 | 说明 | 示例值 |
|------|------|--------|
| `ANTHROPIC_AUTH_TOKEN` | 百炼 API Key | `sk-sp-1f07658367b9409393e075f9f63490bf` |
| `ANTHROPIC_BASE_URL` | 百炼 Anthropic 兼容端点 | `https://coding.dashscope.aliyuncs.com/apps/anthropic` |
| `ANTHROPIC_MODEL` | 默认模型 | `glm-5` |
| `CLAUDE_FALLBACK_MODEL` | 备用模型（主模型失败时） | `qwen3.6-plus` |
| `model` | settings 中的 model 字段 | `glm-5` |
| `permissionMode` | 权限模式 | `bypass`（完全权限）或 `default`（交互式） |
| `effortLevel` | 模型努力程度 | `xhigh`（最高）|

---

## 四、模型选择

### 4.1 推荐模型

| 场景 | 推荐模型 | 说明 |
|------|---------|------|
| **日常编码** | `glm-5` | 中文优秀，代码能力强，速度快 |
| **复杂推理** | `qwen3.6-plus` | 最强推理能力，1M 上下文 |
| **代码生成** | `qwen3-coder-plus` | 专为代码生成优化 |
| **轻量任务** | `glm-4.7` | 快速响应，低成本 |

### 4.2 模型切换

在 `settings.json` 中修改 `ANTHROPIC_MODEL` 和 `model` 字段：

```bash
# 切换到 qwen3.6-plus
cat > ~/.claude/settings.json << 'EOF'
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "sk-sp-YOUR_KEY",
    "ANTHROPIC_BASE_URL": "https://coding.dashscope.aliyuncs.com/apps/anthropic",
    "ANTHROPIC_MODEL": "qwen3.6-plus",
    "CLAUDE_FALLBACK_MODEL": "glm-5"
  },
  "model": "qwen3.6-plus"
}
EOF
```

---

## 五、权限配置

### 5.1 权限模式

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| **bypass** | 所有操作自动允许，无需确认 | 个人使用，信任 Claude Code |
| **default** | 敏感操作需要用户确认 | 生产环境，需要审计 |

### 5.2 推荐权限列表

```json
{
  "permissions": {
    "allow": [
      "Bash(*)",       // 允许执行所有 bash 命令
      "Read",          // 允许读取文件
      "Edit",          // 允许编辑文件
      "Write",         // 允许写入文件
      "WebFetch",      // 允许获取网页内容
      "WebSearch",     // 允许搜索
      "Glob",          // 允许文件匹配
      "Grep",          // 允许内容搜索
      "TodoWrite",     // 允许写入 TODO
      "Agent(*)"       // 允许调用子 Agent
    ]
  }
}
```

---

## 六、高级配置

### 6.1 环境变量

除了 `settings.json`，也可以通过环境变量配置：

```bash
# 添加到 ~/.zshrc 或 ~/.bashrc
export ANTHROPIC_AUTH_TOKEN="sk-sp-YOUR_KEY"
export ANTHROPIC_BASE_URL="https://coding.dashscope.aliyuncs.com/apps/anthropic"
export ANTHROPIC_MODEL="glm-5"
export CLAUDE_FALLBACK_MODEL="qwen3.6-plus"
```

### 6.2 代理配置

如果需要通过代理访问：

```bash
export HTTPS_PROXY=http://127.0.0.1:7897
export HTTP_PROXY=http://127.0.0.1:7897
```

### 6.3 Telegram 集成

启用 Telegram 插件（可选）：

```json
{
  "enabledPlugins": {
    "telegram@claude-plugins-official": true
  }
}
```

---

## 七、常见问题

### 7.1 模型调用失败

**症状**: `401 Unauthorized` 或 `Invalid API Key`

**解决**:
1. 确认 `ANTHROPIC_AUTH_TOKEN` 格式正确（`sk-sp-` 开头）
2. 确认 API Key 未过期
3. 确认百炼账号已开通 Coding Plan

### 7.2 模型响应慢

**症状**: 响应时间超过 30 秒

**解决**:
1. 检查网络延迟
2. 切换到更快的模型（如 `glm-4.7`）
3. 检查百炼服务状态

### 7.3 上下文长度限制

**症状**: 长对话时丢失上下文

**解决**:
1. 使用 `qwen3.6-plus`（1M 上下文）
2. 定期使用 `/compact` 压缩上下文
3. 拆分复杂任务为多个小任务

### 7.4 权限拒绝

**症状**: 执行命令时需要反复确认

**解决**:
1. 设置 `"permissionMode": "bypass"`
2. 或在 `permissions.allow` 中添加具体命令

---

## 八、最佳实践

### 8.1 模型组合策略

```
日常编码 → glm-5（快 + 准）
复杂推理 → qwen3.6-plus（强推理）
代码生成 → qwen3-coder-plus（专精）
备用     → CLAUDE_FALLBACK_MODEL
```

### 8.2 权限安全

- 个人开发：`bypass` 模式（高效）
- 生产环境：`default` 模式（安全）
- 定期审计：检查 `permissions.allow` 列表

### 8.3 成本优化

- Coding Plan 模型免费使用
- 避免使用非 Coding Plan 模型（会产生费用）
- 定期清理不用的会话历史

### 8.4 性能调优

- 设置 `effortLevel: "xhigh"` 获得最佳输出质量
- 使用 `CLAUDE_FALLBACK_MODEL` 确保高可用
- 配置代理减少网络延迟

---

## 九、验证配置

### 9.1 快速测试

```bash
# 启动 Claude Code
claude

# 或在 VS Code 中打开 Claude Code 扩展

# 输入测试命令
claude> 请告诉我你当前使用的模型是什么？
```

### 9.2 检查当前配置

```bash
# 查看配置文件
cat ~/.claude/settings.json

# 查看运行日志
cat ~/.claude/history.jsonl | tail -10
```

---

## 十、参考链接

- [百炼 Coding Plan 文档](https://help.aliyun.com/zh/model-studio/openclaw-coding-plan)
- [Claude Code 官方文档](https://docs.anthropic.com/en/docs/claude-code/overview)
- [百炼模型列表](https://help.aliyun.com/zh/model-studio/getting-started/models)

---

**版本历史**:

| 日期 | 版本 | 说明 |
|------|------|------|
| 2026-05-03 | 1.0 | 初始版本 |
