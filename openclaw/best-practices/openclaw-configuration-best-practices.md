# OpenClaw 配置最佳实践

> 记录 OpenClaw 多 Agent 系统中的配置文件结构和最佳实践

**创建时间**: 2026-03-04  
**适用版本**: OpenClaw v2026.3+  
**标签**: `openclaw` `configuration` `multi-agent` `best-practices`

---

## 📄 配置文件层次结构

OpenClaw 使用两层配置系统：

```
~/.openclaw/
├── openclaw.json              # 全局 Gateway 配置
└── agents/
    ├── main/
    │   ├── config.json        # Agent 级别配置
    │   └── workspace/
    ├── coding/
    │   ├── config.json        # Agent 级别配置
    │   └── workspace/
    └── stock/
        ├── config.json        # Agent 级别配置
        └── workspace/
```

---

## 🔧 config.json - Agent 级别配置

### ✅ 支持的字段

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

| 字段 | 说明 | 是否支持 |
|------|------|---------|
| `sandbox` | Docker 沙箱配置 | ✅ 支持 |
| `tools.allow` | 允许的工具列表 | ✅ 支持 |
| `tools.deny` | 拒绝的工具列表 | ✅ 支持 |
| `tools.elevated` | 提权工具配置 | ✅ 支持 |
| `skills` | 技能列表 | ❌ **不支持** |

### ❌ 常见错误配置

```json
{
  "skills": ["github", "pr-advocacy"]  // ❌ 无效！
}
```

**正确做法**: 技能通过工作区目录控制，而非配置文件
```bash
# 安装到特定 Agent 工作区
clawhub install github --cwd /Users/hope/.openclaw/agents/coding/workspace
```

---

## 🌍 openclaw.json - 全局 Gateway 配置

### 核心字段结构

```json5
{
  // ========== Channels ==========
  "channels": {
    "telegram": { ... },
    "discord": { ... },
    "whatsapp": { ... }
  },
  
  // ========== 模型配置 ==========
  "models": {
    "providers": { ... }
  },
  
  // ========== Agent 配置 ==========
  "agents": {
    "defaults": {
      "model": { "primary": "..." },
      "sandbox": { ... },
      "heartbeat": { ... }
    },
    "list": [
      { 
        "id": "main", 
        "workspace": "...",
        "model": "...",
        "identity": { ... }
      }
    ]
  },
  
  // ========== 路由绑定 ==========
  "bindings": [
    { "agentId": "main", "match": { ... } }
  ],
  
  // ========== 技能配置 ==========
  "skills": {
    "entries": { "skill-name": { "enabled": true } },
    "load": { "extraDirs": [...] }
  }
}
```

---

## 🎯 配置位置决策表

| 配置需求 | 文件位置 | 配置路径 |
|---------|---------|---------|
| 禁用某个技能 | `openclaw.json` | `skills.entries.<name>.enabled: false` |
| Agent 专属技能 | 工作区目录 | `<workspace>/skills/` |
| Agent 沙箱 | `config.json` (优先) 或 `openclaw.json` | `sandbox.mode` |
| Agent 工具权限 | `config.json` (覆盖) 或 `openclaw.json` | `tools.allow/deny` |
| Agent 模型 | `openclaw.json` | `agents.list[].model` |
| Agent 身份 | `openclaw.json` | `agents.list[].identity` |
| Channel 配置 | `openclaw.json` | `channels.<provider>` |
| 路由规则 | `openclaw.json` | `bindings[]` |
| 全局技能目录 | `openclaw.json` | `skills.load.extraDirs` |

---

## 📚 技能加载优先级

```
<workspace>/skills/     (最高优先级 - Agent 专属)
       ↓
~/.openclaw/skills/    (全局共享 - 所有 Agent)
       ↓
   捆绑技能             (系统内置 - 最低优先级)
```

### 最佳实践

1. **全局共享技能** → 安装到 `~/.openclaw/skills/`
   ```bash
   clawhub install self-improvement --cwd ~/.openclaw
   ```

2. **Agent 专属技能** → 安装到工作区
   ```bash
   clawhub install github-contribution --cwd /Users/hope/.openclaw/agents/coding/workspace
   ```

3. **禁用技能** → 全局配置
   ```json
   {
     "skills": {
       "entries": {
         "sag": { "enabled": false }
       }
     }
   }
   ```

---

## 🔐 安全配置建议

### 1. 沙箱隔离

```json
{
  "sandbox": {
    "mode": "non-main",  // main agent 不沙箱，其他 agent 沙箱
    "scope": "agent",    // 每个 agent 独立容器
    "docker": {
      "network": "none", // 默认无网络
      "setupCommand": "apt-get update && apt-get install -y git curl"
    }
  }
}
```

### 2. 工具权限最小化

```json
{
  "tools": {
    "allow": ["read", "write", "edit"],  // 只允许必要工具
    "deny": ["browser", "canvas", "nodes"]  // 拒绝高风险工具
  }
}
```

### 3. 技能访问控制

- 敏感技能只安装到特定 Agent 工作区
- 使用 `skills.entries` 全局禁用不需要的技能
- 定期审查 `~/.openclaw/skills/` 目录

---

## 🛠️ 常见问题排查

### Q1: 技能不生效？

**检查顺序**:
1. 确认技能安装在正确位置
2. 检查 `openclaw.json` 中是否被禁用
3. 验证技能元数据 gating 条件
4. 重启 Gateway 或等待 watcher 刷新

### Q2: config.json 修改不生效？

**可能原因**:
- 修改了不支持的字段（如 `skills`）
- 语法错误（使用 JSON5 格式）
- 需要重启 Gateway

**验证命令**:
```bash
openclaw agents list --bindings
openclaw gateway status
```

### Q3: 如何为不同 Agent 配置不同模型？

**正确做法** (在 `openclaw.json`):
```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "model": "anthropic/claude-sonnet-4-5"
      },
      {
        "id": "coding",
        "model": "anthropic/claude-opus-4-6"
      }
    ]
  }
}
```

---

## 📖 相关文档

- [OpenClaw 官方文档 - Configuration](https://docs.openclaw.ai/gateway/configuration)
- [OpenClaw 官方文档 - Skills](https://docs.openclaw.ai/tools/skills)
- [OpenClaw 官方文档 - Multi-Agent](https://docs.openclaw.ai/concepts/multi-agent)
- [本项目 - GitHub Workflow 最佳实践](./github-workflow-best-practices.md)

---

**最后更新**: 2026-03-04  
**维护者**: OpenClaw Main Agent
