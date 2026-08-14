# OpenClaw Memory Search 开启指南

## 什么是 Memory Search？

OpenClaw Memory Search 是内置的语义搜索功能，可以：
- 自动索引 `MEMORY.md` + `memory/*.md` 文件
- 通过向量搜索实现语义召回（而非简单的文本匹配）
- 在每次对话前自动搜索相关记忆片段

**核心优势：**
- 解决 AI 会话上下文丢失问题
- 重要决策、经验、配置自动召回
- 无需手动翻阅历史文件

---

## 与 memory-lancedb 的区别

| 功能 | memory-core (Memory Search) | memory-lancedb (插件) |
|------|---------------------------|---------------------|
| **索引对象** | 文件（MEMORY.md + memory/*.md） | 对话历史 |
| **触发时机** | 每次会话开始时 | 对话结束后自动捕获 |
| **配置位置** | `agents.defaults.memorySearch` | `plugins.slots.memory` |
| **用途** | 静态知识库搜索 | 动态对话记忆 |

**两者可以并存！** 分别管理文件索引和对话记忆。

---

## 如何开启 Memory Search

### 基本配置

在 `openclaw.json` 中添加：

```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "enabled": true,
        "provider": "openai",
        "model": "text-embedding-3-small"
      }
    }
  }
}
```

### 配置项说明

| 配置项 | 说明 | 必需 |
|-------|------|------|
| `enabled` | 开关 | ✅ |
| `provider` | Embedding 提供商 | ✅ |
| `model` | Embedding 模型名 | 推荐 |
| `remote.baseUrl` | 自定义 API 地址 | 可选 |
| `remote.apiKey` | 自定义 API Key | 可选 |
| `sources` | 索引源（memory/sessions） | 可选，默认 ["memory"] |
| `extraPaths` | 额外索引路径 | 可选 |
| `sync.watch` | 实时监控文件变化 | 可选 |

---

## 不同 Provider 配置示例

### 1. OpenAI（默认）

```json
"memorySearch": {
  "provider": "openai",
  "model": "text-embedding-3-small"
}
```

**模型选择：**
- `text-embedding-3-small` - 推荐，性价比高
- `text-embedding-3-large` - 更高质量
- `text-embedding-ada-002` - 旧版本

### 2. Ollama（本地）

适合无 API 限制、隐私敏感的场景：

```json
"memorySearch": {
  "provider": "openai",
  "remote": {
    "baseUrl": "http://localhost:11434/v1",
    "apiKey": "ollama"
  },
  "model": "nomic-embed-text"
}
```

**前提条件：**
```bash
# 安装 Ollama
brew install ollama

# 拉取 embedding 模型
ollama pull nomic-embed-text

# 确保 Ollama 运行
ollama serve
```

**推荐模型：**
- `nomic-embed-text` - 768 维，性能好
- `all-minilm` - 384 维，轻量级
- `mxbai-embed-large` - 1024 维，高质量

### 3. 百炼（阿里云 DashScope）

适合国内用户，稳定可用：

```json
"memorySearch": {
  "provider": "openai",
  "remote": {
    "baseUrl": "https://dashscope.aliyuncs.com/compatible-mode/v1",
    "apiKey": "YOUR_DASHSCOPE_API_KEY"
  },
  "model": "text-embedding-v2"
}
```

**获取 API Key：**
1. 登录 [百炼控制台](https://bailian.console.aliyun.com/)
2. API-KEY 管理 → 创建

**模型选择：**
- `text-embedding-v1` - 1536 维
- `text-embedding-v2` - 推荐，1024 维
- `text-embedding-v3` - 多语言支持

### 4. Gemini

```json
"memorySearch": {
  "provider": "gemini",
  "model": "text-embedding-004"
}
```

**注意：** Gemini 需在环境变量设置 API Key：
```bash
export GEMINI_API_KEY=YOUR_KEY
```

### 5. Voyage AI

专业 embedding 服务，质量高：

```json
"memorySearch": {
  "provider": "voyage",
  "model": "voyage-3"
}
```

### 6. Local（本地计算）

无需外部 API，但质量较低：

```json
"memorySearch": {
  "provider": "local"
}
```

---

## Docker 容器内配置

容器内访问宿主机服务需使用 `host.docker.internal`：

```json
"memorySearch": {
  "provider": "openai",
  "remote": {
    "baseUrl": "http://host.docker.internal:11434/v1",
    "apiKey": "ollama"
  },
  "model": "nomic-embed-text"
}
```

---

## 高级配置

### 实时监控文件变化

```json
"memorySearch": {
  "sync": {
    "watch": true
  }
}
```

当 `MEMORY.md` 或 `memory/*.md` 修改时，自动重新索引。

### 扩展索引路径

索引其他目录的 .md 文件：

```json
"memorySearch": {
  "extraPaths": ["~/.openclaw/workspace/skills"]
}
```

### Fallback 配置

主 embedding 失败时的备用方案：

```json
"memorySearch": {
  "provider": "openai",
  "fallback": "local"
}
```

---

## 验证配置

### 检查索引状态

```bash
# 查看索引文件（SQLite）
ls -la ~/.openclaw/memory/

# 大小应随 MEMORY.md 增长
```

### 测试搜索

在对话中提及之前记录的内容，AI 应能自动召回：

> "还记得我们在 3 月讨论的 GitHub SSH 配置吗？"

如果 AI 能正确引用 MEMORY.md 中的内容，说明 memory search 正常工作。

---

## 常见问题

### Q: Memory Search 不工作？

**排查步骤：**
1. 检查 `enabled: true` 是否设置
2. 检查 embedding API 是否可用（curl 测试）
3. 检查 `MEMORY.md` 是否存在且有内容
4. 检查模型是否正确（`nomic-embed-text` vs `text-embedding-3-small`）

### Q: Docker 容器无法访问 Ollama？

**解决方案：**
- 使用 `host.docker.internal` 替代 `localhost`
- 或在 Docker run 时添加 `--network host`

### Q: 索引占用空间过大？

**优化方案：**
- 使用较小维度的模型（如 `nomic-embed-text` 768 维）
- 定期清理 `memory/*.md` 旧文件
- 控制 `MEMORY.md` 大小（建议 < 10KB）

---

## 最佳实践

1. **保持 MEMORY.md 精简** - 只记录真正重要的内容
2. **定期归档** - 旧内容移到 `memory/YYYY-MM-DD.md`
3. **选择可靠的 Provider** - 生产环境用 OpenAI/百炼，本地测试用 Ollama
4. **启用 Fallback** - 保证服务可靠性
5. **开启实时监控** - `sync.watch: true`

---

## 完整配置示例

```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "enabled": true,
        "provider": "openai",
        "remote": {
          "baseUrl": "http://localhost:11434/v1",
          "apiKey": "ollama"
        },
        "model": "nomic-embed-text",
        "sources": ["memory"],
        "sync": {
          "watch": true
        },
        "fallback": "local"
      }
    }
  },
  "plugins": {
    "slots": {
      "memory": "memory-core"
    },
    "entries": {
      "memory-core": {
        "enabled": true
      }
    }
  }
}
```

---

## 参考链接

- [OpenClaw 官方文档](https://docs.openclaw.ai)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [Ollama](https://ollama.ai)
- [百炼控制台](https://bailian.console.aliyun.com/)