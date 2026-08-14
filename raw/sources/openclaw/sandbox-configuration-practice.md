# OpenClaw 沙箱配置实践指南

## 背景
OpenClaw 默认启用沙箱模式，限制 Agent 对文件系统的访问权限。这对于安全很重要，但在某些场景下（如需要操作项目目录 `/Users/hope/IdeaProjects`），我们需要为特定 Agent 关闭沙箱限制。

## 解决方案

### 1. Agent 级配置
在对应 Agent 的配置文件中关闭沙箱并授予完整工具权限：

**配置文件路径**: `~/.openclaw/agents/{agentId}/config.json`

**配置内容**:
```json
{
  "sandbox": {
    "mode": "off"
  },
  "tools": {
    "allow": ["read", "write", "edit", "exec", "process"],
    "deny": []
  }
}
```

### 2. 配置说明
- **`sandbox: { mode: "off" }`**: 完全关闭 Docker 沙箱隔离
- **`tools.allow`**: 明确允许所有文件操作相关工具
- **`tools.deny: []`**: 不拒绝任何工具

### 3. 工具使用策略
即使配置了完整权限，**`write`/`read`/`edit` 工具仍可能受限于 workspace 路径检查**。推荐使用以下方式：

✅ **推荐**: 使用 `exec` 工具执行 shell 命令
```bash
echo "content" > /path/outside/workspace/file
cp /source/file /destination/file  
mv /old/path /new/path
```

❌ **不推荐**: 依赖 `write` 工具操作外部路径（可能失败）

### 4. 多 Agent 策略
- **main/coding agents**: 关闭沙箱，用于开发和文件操作
- **stock/mcs agents**: 保持沙箱模式，用于安全敏感操作
- **public agents**: 严格沙箱 + 工具限制

### 5. 安全考虑
关闭沙箱后 Agent 将拥有完整的文件系统读写权限，请确保：
- 只对可信的 Agent 关闭沙箱
- 定期审计 Agent 操作日志
- 避免在生产环境随意关闭沙箱

## 验证步骤
1. 创建配置文件
2. 使用 `exec` 工具测试外部文件操作
3. 验证文件创建/修改成功

## 参考文档
- [Full access (no sandbox)](https://docs.openclaw.ai/gateway/configuration-reference#full-access-no-sandbox)
- [Multi-Agent Sandbox & Tools](https://docs.openclaw.ai/tools/multi-agent-sandbox-tools)
