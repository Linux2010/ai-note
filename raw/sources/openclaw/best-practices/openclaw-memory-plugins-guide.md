# OpenClaw Memory 插件对比指南

> 深入解析 memory-core 与 memory-lancedb 两个记忆插件的区别、适用场景及配置方法

**创建时间**: 2026-04-04  
**适用版本**: OpenClaw v2026.3+  
**标签**: `openclaw` `memory` `plugin` `lancedb` `best-practices`

---

## 📋 快速结论

| 对比项 | memory-core | memory-lancedb |
|--------|-------------|----------------|
| **推荐度** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ **推荐** |
| **文件索引** | ✅ 支持 | ✅ 支持 |
| **对话记忆** | ❌ 不支持 | ✅ 支持 |
| **工具数量** | 2 个 | 5 个 |
| **数据存储** | SQLite | SQLite + LanceDB |
| **autoCapture** | ❌ 无 | ✅ 自动捕获 |
| **autoRecall** | ❌ 无 | ✅ 自动召回 |

**一句话总结**: `memory-lancedb` 功能更全面，包含 `memory-core` 的所有功能，额外支持对话记忆。**推荐使用 memory-lancedb**。

---

## 🔍 详细对比

### 1. 功能矩阵

| 功能 | memory-core | memory-lancedb | 说明 |
|------|:-----------:|:-------------:|------|
| `memory_search` | ✅ | ✅ | 搜索 `MEMORY.md` 和 `memory/*.md` 文件 |
| `memory_get` | ✅ | ✅ | 读取特定记忆文件 |
| `memory_recall` | ❌ | ✅ | 搜索 LanceDB 中的对话记忆 |
| `memory_store` | ❌ | ✅ | 手动存储记忆到 LanceDB |
| `memory_forget` | ❌ | ✅ | 删除 LanceDB 中的记忆 |

### 2. 数据存储架构

#### memory-core
```
~/.openclaw/memory/
└── main.sqlite          # 文件索引数据库
    ├── chunks           # 文件内容分块
    ├── files            # 文件路径记录
    └── embedding_cache  # 向量缓存
```

#### memory-lancedb
```
~/.openclaw/memory/
├── main.sqlite          # 文件索引数据库（同上）
└── lancedb/             # 对话记忆数据库
    └── memories.lance/  # LanceDB 表
        └── data/*.parquet
```

### 3. 工作原理对比

#### 文件索引（两者相同）
- 索引 `MEMORY.md` 和 `memory/*.md` 文件
- 使用 Embedding 模型生成向量
- 支持 BM25 + 向量混合搜索
- 文件变更时自动同步（需配置 `sync.watch`）

#### 对话记忆（仅 memory-lancedb）
- **autoCapture**: 对话结束时自动捕获重要信息
  - 检测关键词："记住"、"决定"、"重要"、"偏好"等
  - 提取关键内容存入 LanceDB
- **autoRecall**: 对话开始时自动召回相关记忆
  - 根据当前上下文搜索历史记忆
  - 将相关记忆注入对话上下文

---

## ⚙️ 配置方法

### 方案 A: 使用 memory-core（基础版）

```json
{
  "plugins": {
    "slots": {
      "memory": "memory-core"
    }
  },
  "agents": {
    "defaults": {
      "memorySearch": {
        "provider": "openai",
        "model": "nomic-embed-text",
        "remote": {
          "baseUrl": "http://localhost:11434/v1",
          "apiKey": "ollama"
        },
        "sync": {
          "watch": true
        }
      }
    }
  }
}
```

**适用场景**: 只需要文件索引，不需要对话记忆功能。

---

### 方案 B: 使用 memory-lancedb（推荐）

```json
{
  "plugins": {
    "slots": {
      "memory": "memory-lancedb"
    },
    "entries": {
      "memory-lancedb": {
        "enabled": true,
        "config": {
          "embedding": {
            "provider": "openai",
            "apiKey": "ollama",
            "model": "nomic-embed-text",
            "baseUrl": "http://localhost:11434/v1",
            "dimensions": 768
          },
          "dbPath": "~/.openclaw/memory/lancedb",
          "autoCapture": true,
          "autoRecall": true,
          "captureMaxChars": 500
        }
      }
    }
  },
  "agents": {
    "defaults": {
      "memorySearch": {
        "provider": "openai",
        "model": "nomic-embed-text",
        "remote": {
          "baseUrl": "http://localhost:11434/v1",
          "apiKey": "ollama"
        },
        "sync": {
          "watch": true
        }
      }
    }
  }
}
```

**适用场景**: 需要完整的记忆系统，包括文件索引和对话记忆。

---

## 🔧 配置参数详解

### memorySearch（文件索引配置）

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `provider` | Embedding 提供商 | `"openai"`（兼容端点） |
| `model` | Embedding 模型 | `"nomic-embed-text"`（本地）或 `"text-embedding-3-small"`（云端） |
| `remote.baseUrl` | API 地址 | `"http://localhost:11434/v1"`（Ollama） |
| `remote.apiKey` | API Key | `"ollama"`（Ollama）或实际 Key |
| `sync.watch` | 文件监听 | `true`（自动同步） |

### memory-lancedb（对话记忆配置）

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `embedding.dimensions` | 向量维度 | `768`（nomic）或 `1536`（OpenAI） |
| `autoCapture` | 自动捕获 | `true` |
| `autoRecall` | 自动召回 | `true` |
| `captureMaxChars` | 最大捕获字符 | `500` |

---

## ⚠️ 常见配置错误

### 错误 1: Provider 配置错误

```json
// ❌ 错误 - Ollama 兼容端点不能用 provider: "ollama"
"memorySearch": {
  "provider": "ollama"
}

// ✅ 正确 - OpenAI 兼容端点必须用 provider: "openai"
"memorySearch": {
  "provider": "openai",
  "remote": { "baseUrl": "http://localhost:11434/v1" }
}
```

### 错误 2: 同时配置两个插件

```json
// ❌ 错误 - slot 只能有一个插件
"plugins": {
  "slots": {
    "memory": "memory-core"
  },
  "entries": {
    "memory-lancedb": { "enabled": true }  // 会被警告
  }
}
```

**Doctor 警告**: 
> `plugins.entries.memory-lancedb: plugin disabled (memory slot set to "memory-core") but config is present`

### 错误 3: 维度不匹配

```json
// ❌ 错误 - 维度与模型不匹配
"embedding": {
  "model": "nomic-embed-text",  // 768 维
  "dimensions": 1536            // ❌ 不匹配
}

// ✅ 正确
"embedding": {
  "model": "nomic-embed-text",
  "dimensions": 768
}
```

---

## 🛠️ 验证配置

### 检查插件状态

```bash
openclaw doctor --non-interactive
```

### 检查索引状态

```bash
sqlite3 ~/.openclaw/memory/main.sqlite "SELECT COUNT(*) FROM chunks;"
```

### 检查对话记忆

```bash
ls ~/.openclaw/memory/lancedb/memories.lance/data/*.parquet | wc -l
```

---

## 📖 相关文档

- [OpenClaw 官方文档 - Memory](https://docs.openclaw.ai/concepts/memory)
- [OpenClaw 官方文档 - Plugins](https://docs.openclaw.ai/plugins/sdk-overview)
- [OpenClaw CLI - Memory](https://docs.openclaw.ai/cli/memory)
- [LanceDB 官方文档](https://lancedb.github.io/lancedb/)

---

## 🔄 迁移指南

### 从 memory-core 迁移到 memory-lancedb

1. 修改配置：
   ```json
   "plugins": {
     "slots": { "memory": "memory-lancedb" }
   }
   ```

2. 添加 memory-lancedb 配置（见上方）

3. 重启 Gateway：
   ```bash
   # 配置会自动热加载，无需重启
   # 如需强制刷新：
   openclaw gateway restart
   ```

4. 验证：
   ```bash
   openclaw doctor --non-interactive
   ```

**注意**: 文件索引数据（`main.sqlite`）会保留，无需重新索引。

---

**最后更新**: 2026-04-04  
**维护者**: OpenClaw Main Agent  
**贡献者**: Andy (hope)