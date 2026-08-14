# OpenClaw Memory Search 技术实现

> **最后更新**: 2026-04-01  
> **适用版本**: OpenClaw v0.10+  
> **标签**: #memory #vector-search #sqlite #embedding

---

## 概述

OpenClaw 内置了强大的**混合记忆搜索系统**，结合了向量相似度搜索和全文关键词搜索 (BM25)，为 AI Agent 提供语义级别的记忆检索能力。

---

## 核心架构

```
┌─────────────────────────────────────────────────────────┐
│  数据源 (Source Files)                                   │
│  ~/.openclaw/agents/{agent}/workspace/MEMORY.md         │
│  ~/.openclaw/agents/{agent}/workspace/memory/*.md       │
└────────────────────┬────────────────────────────────────┘
                     │ 同步/索引 (自动)
                     ↓
┌─────────────────────────────────────────────────────────┐
│  向量数据库 (SQLite + sqlite-vec 扩展)                    │
│  ~/.openclaw/memory/{agentId}.sqlite                    │
│  ├─ chunks (文本分块元数据)                              │
│  ├─ chunks_vec (向量索引表)                              │
│  ├─ chunks_vec_vector_chunks00 (实际向量数据)            │
│  └─ chunks_fts* (FTS5 全文搜索索引)                       │
└────────────────────┬────────────────────────────────────┘
                     │ 查询
                     ↓
┌─────────────────────────────────────────────────────────┐
│  memory_search 工具                                      │
│  1. Ollama 生成查询向量 (nomic-embed-text)              │
│  2. SQLite 混合搜索 (向量 + FTS)                         │
│  3. 加权融合 + MMR 去重                                  │
│  4. 返回结果 (带原始文件路径引用)                          │
└─────────────────────────────────────────────────────────┘
```

---

## 技术栈

| 组件 | 技术 | 用途 |
|------|------|------|
| **向量存储** | SQLite + sqlite-vec | 存储和查询向量嵌入 |
| **全文搜索** | SQLite FTS5 | BM25 关键词搜索 |
| **嵌入模型** | Ollama (`nomic-embed-text`) | 生成文本向量 |
| **混合策略** | 加权融合 + MMR | 平衡相关性和多样性 |

---

## 混合搜索流程

### 1. 查询向量化

```typescript
// 使用 Ollama 生成查询向量
const queryVec = await ollama.embed({
  model: "nomic-embed-text",
  text: userQuery
});
```

### 2. 并行搜索

```typescript
// 向量搜索 (余弦相似度)
const vectorResults = await searchVector({
  db: sqliteDb,
  vectorTable: "chunks_vec",
  queryVec: queryVec,
  limit: maxResults * candidateMultiplier  // 扩大候选
});

// 关键词搜索 (BM25)
const keywordResults = await searchKeyword({
  db: sqliteDb,
  ftsTable: "chunks_fts",
  query: userQuery,
  limit: maxResults * candidateMultiplier
});
```

### 3. 分数归一化与融合

```typescript
// 归一化分数到 [0, 1]
const normalizedVector = normalizeScore(vectorResults);
const normalizedText = normalizeScore(keywordResults);

// 加权融合
const finalScore = 
  vectorWeight * normalizedVector + 
  textWeight * normalizedText;
// 默认：vectorWeight=0.7, textWeight=0.3
```

### 4. MMR 去重 (可选)

```typescript
// Maximal Marginal Relevance
const reranked = mmrRerank(results, {
  enabled: true,
  lambda: 0.7  // 0=最大多样性，1=最大相关性
});
```

---

## 配置参数

### 全局配置 (`~/.openclaw/openclaw.json`)

```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "enabled": true,
        "provider": "ollama",
        "query": {
          "maxResults": 6,
          "minScore": 0.35,
          "hybrid": {
            "enabled": true,
            "vectorWeight": 0.7,
            "textWeight": 0.3,
            "candidateMultiplier": 4,
            "mmr": {
              "enabled": false,
              "lambda": 0.7
            },
            "temporalDecay": {
              "enabled": false,
              "halfLifeDays": 30
            }
          }
        }
      }
    }
  }
}
```

### 参数说明

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `enabled` | `true` | 是否启用记忆搜索 |
| `provider` | `"ollama"` | 嵌入模型提供商 |
| `maxResults` | `6` | 返回结果数量 |
| `minScore` | `0.35` | 最低分数阈值 |
| `vectorWeight` | `0.7` | 向量搜索权重 |
| `textWeight` | `0.3` | 关键词搜索权重 |
| `candidateMultiplier` | `4` | 候选扩大倍数 |
| `mmr.enabled` | `false` | 是否启用 MMR 去重 |
| `mmr.lambda` | `0.7` | MMR 相关性/多样性平衡 |

---

## 多 Agent 支持

### 独立数据库

每个 Agent 有独立的向量数据库：

```
~/.openclaw/memory/
├── main.sqlite      # Main Agent
├── stock.sqlite     # Stock Agent
├── coding.sqlite    # Coding Agent
└── ...
```

### 配置继承

```json
{
  "agents": {
    "defaults": {
      "memorySearch": { "enabled": true }  // 全局配置
    },
    "list": [
      {
        "id": "main",
        "memorySearch": { 
          "enabled": true,
          "query": { "hybrid": { "vectorWeight": 0.8 } }  // 覆盖
        }
      },
      {
        "id": "stock",
        // 继承 defaults 配置
      }
    ]
  }
}
```

---

## 使用示例

### 工具调用

```typescript
// 在 Agent 中调用 memory_search
const results = await memory_search({
  query: "frp 备份配置",
  maxResults: 5,
  minScore: 0.35
});

// 返回结果
{
  "results": [
    {
      "path": "MEMORY.md",
      "startLine": 1,
      "endLine": 32,
      "score": 0.424,
      "snippet": "# Main Agent - Core Memory...",
      "citation": "MEMORY.md#L1-L32"
    }
  ],
  "provider": "ollama",
  "model": "nomic-embed-text",
  "mode": "hybrid"
}
```

### 读取记忆文件

```typescript
// 根据搜索结果读取完整内容
const content = await memory_get({
  path: "MEMORY.md",
  from: 1,
  lines: 32
});
```

---

## 性能优化

### 1. 批量嵌入

```json
{
  "remote": {
    "batch": {
      "enabled": true,
      "concurrency": 2,
      "wait": true,
      "pollIntervalMs": 2000
    }
  }
}
```

### 2. 缓存

```json
{
  "cache": {
    "enabled": true,
    "maxEntries": 1000
  }
}
```

### 3. 同步策略

```json
{
  "sync": {
    "onSessionStart": true,
    "onSearch": true,
    "watch": true,
    "watchDebounceMs": 1500,
    "intervalMinutes": 0
  }
}
```

---

## 故障排查

### 常见问题

#### 1. 搜索结果为空

**原因**: 
- 记忆文件未被索引
- 分数阈值过高
- 嵌入模型不可用

**解决**:
```bash
# 检查 Ollama 状态
ollama list

# 降低 minScore
"minScore": 0.25

# 手动触发同步
# (重启 Agent 或修改记忆文件)
```

#### 2. 向量搜索慢

**原因**:
- 数据库过大
- 未使用索引

**解决**:
```sql
-- 检查索引
SELECT name FROM sqlite_master 
WHERE type='index' AND tbl_name='chunks_vec';

-- 优化数据库
VACUUM;
```

#### 3. 嵌入失败

**检查**:
```bash
# 测试 Ollama
ollama run nomic-embed-text "test"

# 检查配置
cat ~/.openclaw/openclaw.json | jq '.agents.defaults.memorySearch'
```

---

## 最佳实践

### 1. 记忆文件组织

```
workspace/
├── MEMORY.md              # 核心长期记忆 (COLD)
└── memory/
    ├── hot/
    │   └── HOT_MEMORY.md  # 活跃任务 (HOT)
    ├── warm/
    │   └── WARM_MEMORY.md # 稳定配置 (WARM)
    └── archive/           # 归档记忆
```

### 2. 定期清理

使用 `memory-hygiene` 技能定期清理无效记忆：

```bash
openclaw run memory-hygiene --agent main
```

### 3. 分层记忆

- **HOT**: 当前任务，频繁访问
- **WARM**: 稳定配置，偶尔更新
- **COLD**: 核心记忆，长期保存

---

## 参考资料

- [OpenClaw 官方文档](https://docs.openclaw.ai)
- [sqlite-vec](https://github.com/asg017/sqlite-vec)
- [SQLite FTS5](https://www.sqlite.org/fts5.html)
- [MMR 论文](https://www.cs.cmu.edu/~jgc/publication/The_Use_MMR_Diversity_Based_LTMIR_1998.pdf)

---

*本文档由 AI 生成并维护，欢迎提交 PR 贡献更多实践经验。*
