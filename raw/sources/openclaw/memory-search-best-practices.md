# OpenClaw Memory-Search 插件完全指南

> **最后更新**: 2026-04-09  
> **适用版本**: OpenClaw 2026.4.2+  
> **作者**: AI-Note Community

---

## 📊 执行摘要

OpenClaw 提供 **4 种 memory 插件方案**，每种适用于不同场景：

| 插件 | 推荐度 | 适用场景 | 难度 |
|------|--------|---------|------|
| **Memory Core (Builtin)** | ⭐⭐⭐⭐⭐ | 默认选择，90% 用户 | 简单 |
| **QMD** | ⭐⭐⭐⭐ | 需要 rerank/搜索外部文件 | 中等 |
| **Honcho** | ⭐⭐⭐⭐ | 跨会话记忆 + 用户建模 | 中等 |
| **LanceDB Pro** | ⭐⭐⭐⭐ | 自动捕获 + 高性能检索 | 复杂 |
| **LanceDB (基础版)** | ⭐⭐ | 简单向量存储 | 中等 |

**快速推荐**：
- 🟢 **新手/大多数用户** → Memory Core（你正在用的）
- 🟡 **需要更强检索** → Memory Core + QMD
- 🟠 **跨 Agent 协作** → Memory Core + Honcho
- 🔴 **特殊需求** → LanceDB Pro

---

## 🔍 插件详细对比

### 1. Memory Core (Builtin) - 默认推荐

**插件名称**: `memory-core`  
**状态**: ✅ 内置（默认启用）

#### 核心特性

| 特性 | 说明 |
|------|------|
| **存储格式** | Markdown 文件（人类可读） |
| **索引后端** | SQLite + FTS5 + sqlite-vec |
| **搜索方式** | 混合搜索（向量 + 关键词） |
| **工具** | `memory_search`, `memory_get` |
| **自动索引** | ✅ 1.5s 防抖 |
| **依赖** | 无 |

#### 技术架构

```
MEMORY.md + memory/*.md
         ↓ (自动索引)
~/.openclaw/memory/<agentId>.sqlite
         ↓ (混合搜索)
├── FTS5 全文索引 (BM25)
└── sqlite-vec 向量索引 (余弦相似度)
```

#### 配置示例

```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "enabled": true,
        "provider": "ollama",
        "sync": {
          "watch": true,
          "watchDebounceMs": 1500
        },
        "query": {
          "maxResults": 8,
          "hybrid": {
            "enabled": true,
            "vectorWeight": 0.7,
            "textWeight": 0.3
          }
        }
      }
    }
  }
}
```

#### 优点 ✅

- 开箱即用，无需额外依赖
- Markdown 文件人类可读、可编辑
- 支持所有 embedding provider
- 混合搜索效果好
- CJK（中日韩）支持良好

#### 缺点 ❌

- 无 reranking（重排序）
- 无法索引 workspace 外文件
- 无会话转录索引

#### 适用场景

✅ **推荐使用**：
- 个人知识库管理
- 长期记忆存储
- 需要人类可读的记忆文件
- 简单部署（无额外依赖）

❌ **不推荐**：
- 需要搜索项目文档（workspace 外）
- 需要会话历史检索
- 需要 reranking 提升检索质量

---

### 2. QMD Memory Engine - 增强检索

**插件名称**: `qmd` (通过 `memory.backend` 配置)  
**状态**: ✅ 官方支持

#### 核心特性

| 特性 | 说明 |
|------|------|
| **存储格式** | Markdown 文件 + QMD 索引 |
| **索引后端** | QMD Sidecar (Bun + node-llama-cpp) |
| **搜索方式** | BM25 + 向量 + Rerank |
| **工具** | `memory_search`, `memory_get` |
| **自动索引** | ✅ 5 分钟周期更新 |
| **依赖** | QMD binary (~2GB GGUF 模型) |

#### 技术架构

```
MEMORY.md + memory/*.md + 外部目录
         ↓ (qmd update + qmd embed)
~/.openclaw/agents/<agentId>/qmd/
         ↓ (混合搜索 + Rerank)
├── BM25 关键词搜索
├── 向量搜索 (GGUF 本地模型)
└── Cross-Encoder Rerank
```

#### 配置示例

```json
{
  "memory": {
    "backend": "qmd",
    "qmd": {
      "paths": [
        {
          "name": "docs",
          "path": "~/notes",
          "pattern": "**/*.md"
        }
      ],
      "sessions": {
        "enabled": true
      }
    }
  }
}
```

#### 优点 ✅

- **Reranking** 提升检索质量
- **Query Expansion** 查询扩展
- 可索引 workspace 外文件
- 可索引会话转录
- 完全本地运行（无 API Key）
- 自动 fallback 到 Builtin

#### 缺点 ❌

- 需要安装 QMD binary
- 首次搜索慢（下载 ~2GB 模型）
- 配置复杂度增加
- 占用更多磁盘空间

#### 适用场景

✅ **推荐使用**：
- 需要搜索项目文档
- 需要会话历史检索
- 需要更高检索质量（rerank）
- 完全本地化部署

❌ **不推荐**：
- 简单个人使用
- 磁盘空间有限
- 不想安装额外依赖

---

### 3. Honcho Memory - 跨会话记忆

**插件名称**: `@honcho-ai/openclaw-honcho`  
**状态**: ✅ 第三方插件（官方推荐）

#### 核心特性

| 特性 | 说明 |
|------|------|
| **存储格式** | Honcho Service (本地或云端) |
| **索引后端** | Honcho 专用数据库 |
| **搜索方式** | 语义搜索 + 用户建模 |
| **工具** | `honcho_context`, `honcho_ask`, `honcho_search_*` |
| **自动捕获** | ✅ 每轮对话后持久化 |
| **依赖** | Honcho 插件 + Service |

#### 技术架构

```
每轮对话 → Honcho Service
           ↓
    ┌──────┴──────┐
    │  User Model │ ← 用户画像（偏好、事实、风格）
    │ Agent Model │ ← Agent 画像（性格、行为）
    │  Sessions   │ ← 跨会话记忆
    └─────────────┘
           ↓
    对话前注入相关上下文
```

#### 配置示例

```json
{
  "plugins": {
    "entries": {
      "openclaw-honcho": {
        "config": {
          "apiKey": "your-api-key",
          "workspaceId": "openclaw",
          "baseUrl": "https://api.honcho.dev"
        }
      }
    }
  }
}
```

#### 优点 ✅

- **跨会话记忆** 自动持久化
- **用户建模** 自动维护用户画像
- **多 Agent 感知** 父子 Agent 追踪
- 可与 Memory Core 共存
- 支持自托管

#### 缺点 ❌

- 需要额外插件安装
- 需要 Honcho Service（本地或云端）
- 学习曲线较陡
- 与 Markdown 文件不同步

#### 适用场景

✅ **推荐使用**：
- 多 Agent 协作场景
- 需要跨会话记忆
- 需要用户建模
- 企业级部署

❌ **不推荐**：
- 单 Agent 简单使用
- 不想维护额外服务
- 偏好 Markdown 文件

---

### 4. LanceDB Pro Memory - 自动捕获 + 高性能检索

**插件名称**: `memory-lancedb-pro`  
**状态**: ✅ 社区插件（活跃维护）  
**官方文档**: https://github.com/CortexReach/memory-lancedb-pro

#### 核心特性

| 特性 | 说明 |
|------|------|
| **存储格式** | LanceDB 向量数据库 |
| **索引后端** | LanceDB + FTS5 + Cross-Encoder |
| **搜索方式** | 向量 + BM25 + Rerank (RRF 融合) |
| **工具** | `memory_recall`, `memory_store`, `memory_forget`, `memory-pro` CLI |
| **自动捕获** | ✅ autoCapture + smartExtraction (LLM 6 分类) |
| **自动遗忘** | ✅ Weibull 衰减模型 |
| **依赖** | LanceDB binary + Node.js + AVX/AVX2 CPU |

#### 技术架构

```
对话内容 → autoCapture → smartExtraction (LLM 分类)
                        ↓
              ┌─────────┴─────────┐
              │ 6 类别分类：       │
              │ - Profiles        │
              │ - Preferences     │
              │ - Entities        │
              │ - Events          │
              │ - Cases           │
              │ - Patterns        │
              └─────────┬─────────┘
                        ↓
              LanceDB 向量数据库
              (支持 AVX/AVX2 指令集)
                        ↓
              混合检索引擎：
              1. Vector + BM25 (RRF 融合)
              2. Cross-Encoder Rerank
              3. Recency Boost
              4. Time Decay
              5. MMR Diversity
                        ↓
              memory_recall / auto-recall
```

#### 配置示例

```json
{
  "plugins": {
    "slots": {
      "memory": "memory-lancedb-pro"
    },
    "entries": {
      "memory-lancedb-pro": {
        "enabled": true,
        "config": {
          "embedding": {
            "provider": "openai-compatible",
            "apiKey": "${OPENAI_API_KEY}",
            "model": "text-embedding-3-small"
          },
          "autoCapture": true,
          "autoRecall": true,
          "smartExtraction": true,
          "extractMinMessages": 2,
          "extractMaxChars": 8000,
          "retrieval": {
            "mode": "hybrid",
            "vectorWeight": 0.7,
            "bm25Weight": 0.3,
            "candidatePoolSize": 12,
            "minScore": 0.6,
            "hardMinScore": 0.62
          },
          "rerank": {
            "enabled": true,
            "provider": "jina",
            "model": "jina-reranker-v2-base-multilingual"
          },
          "sessionMemory": {
            "enabled": false
          }
        }
      }
    }
  }
}
```

#### 优点 ✅

- **自动捕获** - 无需手动 `memory_store`，从对话自动学习
- **智能提取** - LLM 驱动的 6 类别分类
- **智能遗忘** - Weibull 衰减模型，重要记忆保留，噪音自然消失
- **混合检索** - Vector + BM25 + Cross-Encoder Rerank
- **多范围隔离** - Per-agent/per-user/per-project 记忆边界
- **完整工具链** - CLI, backup, migration, export/import
- **任何 Provider** - OpenAI, Jina, Gemini, Ollama 等

#### 缺点 ❌

- **不存储 Markdown** - LanceDB 二进制存储，人类不可读
- **CPU 要求** - 需要 AVX/AVX2 指令集 (2011+ CPU)
- **与 Memory Core 互斥** - 不能同时使用
- **不与 Markdown 同步** - `MEMORY.md` 内容不会被 `memory_recall` 检索
- **复杂度** - 配置参数多，学习曲线陡
- **依赖多** - 需要 LanceDB binary + Node.js

#### 适用场景

✅ **推荐使用**：
- 需要完全自动化的记忆捕获
- 不需要人类可读的记忆文件
- 需要高性能混合检索
- 需要智能遗忘机制

❌ **不推荐**：
- 需要 Markdown 文件（人类可读）
- 需要手动编辑记忆
- CPU 不支持 AVX/AVX2
- 简单个人使用

#### 重要注意事项

⚠️ **双重记忆层不同步**：

当 `memory-lancedb-pro` 激活时，系统有**两个独立的记忆层**：

| 记忆层 | 存储 | 用途 | 可检索？ |
|--------|------|------|---------|
| Plugin Memory | LanceDB | `memory_recall` / auto-recall | ✅ |
| Markdown Memory | `MEMORY.md`, `memory/*.md` | 启动上下文，人类可读日记 | ❌ |

**关键原则**：
> 写入 `memory/YYYY-MM-DD.md` 的事实会在启动时可见，但 `memory_recall` **不会**找到它，除非它也是通过 `memory_store` 写入或被插件自动捕获的。

**这意味着**：
- 需要语义检索？→ 使用 `memory_store` 或让 auto-capture 完成
- `memory/YYYY-MM-DD.md` → 视为日记/日志，**不是**检索源
- `MEMORY.md` → 人类可读参考，**不是**检索源
- Plugin memory → `memory_recall` 和 auto-recall 的主要检索源

---

### 5. LanceDB (基础版) - 简单向量存储

**插件名称**: `memory-lancedb`  
**状态**: ⚠️ 官方基础版（功能有限）

#### 核心特性

| 特性 | 说明 |
|------|------|
| **存储格式** | LanceDB 向量数据库 |
| **索引后端** | LanceDB 向量索引 |
| **搜索方式** | 向量搜索 |
| **工具** | `memory_recall`, `memory_store`, `memory_forget` |
| **自动捕获** | ✅ autoCapture (无 smartExtraction) |
| **依赖** | LanceDB binary |

#### 与 LanceDB Pro 的区别

| 特性 | 基础版 | Pro 版 |
|------|--------|--------|
| 混合检索 | ❌ | ✅ Vector + BM25 |
| Rerank | ❌ | ✅ Cross-Encoder |
| 智能提取 | ❌ | ✅ LLM 6 分类 |
| 智能遗忘 | ❌ | ✅ Weibull 衰减 |
| CLI 工具 | 基础 | 完整 |
| 维护状态 | ⚠️ 缓慢 | ✅ 活跃 |

**推荐**: 直接使用 **LanceDB Pro**，基础版功能有限且不活跃维护。

#### 核心特性

| 特性 | 说明 |
|------|------|
| **存储格式** | LanceDB 二进制向量 |
| **索引后端** | LanceDB 向量数据库 |
| **搜索方式** | 向量搜索 + BM25 + Rerank |
| **工具** | `memory_recall`, `memory_store`, `memory_forget` |
| **自动捕获** | ✅ autoCapture |
| **依赖** | LanceDB binary + 插件 |

#### 技术架构

```
对话 → autoCapture → LanceDB
                      ↓
              ┌───────┴───────┐
              │ 向量索引      │
              │ BM25 索引     │
              │ Cross-Encoder │
              └───────────────┘
                      ↓
              memory_recall 检索
```

#### 配置示例

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
            "model": "text-embedding-3-small"
          },
          "autoRecall": true,
          "autoCapture": true
        }
      }
    }
  }
}
```

#### 优点 ✅

- 高性能向量搜索
- 自动捕获对话
- 支持 reranking（Pro 版）
- 细粒度记忆管理

#### 缺点 ❌

- **不存储 Markdown**（黑盒）
- 需要额外插件安装
- 平台兼容性问题（macOS ARM）
- 与 Memory Core 互斥
- 难以调试

#### 适用场景

✅ **推荐使用**：
- 需要高性能向量搜索
- 不需要人类可读记忆
- 完全自动化场景

❌ **不推荐**：
- 需要 Markdown 文件
- 需要手动编辑记忆
- 简单个人使用

---

## 📈 性能对比

### 检索质量 benchmark（基于 LOCOMO 数据集）

| 插件 | Recall@5 | Recall@10 | MRR | 提升 |
|------|----------|-----------|-----|------|
| Memory Core | 0.72 | 0.81 | 0.65 | 基线 |
| QMD | 0.78 | 0.86 | 0.71 | +8% |
| LanceDB Pro | 0.82 | 0.89 | 0.75 | +15% |
| LanceDB (基础) | 0.70 | 0.79 | 0.63 | -3% |
| Honcho | 0.74 | 0.83 | 0.67 | +3% |

*数据来源：https://www.lancedb.com/blog/openclaw-memory-from-zero-to-lancedb-pro*

**关键发现**：
- LanceDB Pro 的 **Cross-Encoder Rerank** 提升最显著 (+15%)
- QMD 的 reranking 也有显著提升 (+8%)
- Memory Core 作为基线已经提供 90% 的效果
- LanceDB 基础版无 rerank，效果略差于 Memory Core

### 响应时间对比

| 插件 | 首次搜索 | 后续搜索 | 索引更新 |
|------|---------|---------|---------|
| Memory Core | ~100ms | ~50ms | 1.5s (防抖) |
| QMD | ~5s* | ~500ms | 5min 周期 | ~500MB |
| LanceDB Pro | ~300ms | ~150ms | 实时 | ~200MB |
| LanceDB (基础) | ~200ms | ~100ms | 实时 | ~150MB |
| Honcho | ~300ms | ~150ms | 每轮对话 |

*QMD 首次搜索需要下载 GGUF 模型（~2GB）

---

## 🎯 最佳实践

### 1. 文件组织最佳实践

```
workspace/
├── MEMORY.md              # 长期记忆（持久化事实、偏好、决策）
├── memory/
│   ├── YYYY-MM-DD.md      # 每日笔记（自动加载今天 + 昨天）
│   ├── projects/          # 项目相关记忆
│   │   ├── project-a.md
│   │   └── project-b.md
│   ├── knowledge/         # 知识点整理
│   │   ├── meetings.md
│   │   └── decisions.md
│   └── reference/         # 参考资料
│       ├── contacts.md
│       └── passwords.md
└── DREAMS.md              # （可选）梦境日记
```

### 2. 混合搜索调优

```json
{
  "memorySearch": {
    "query": {
      "hybrid": {
        "enabled": true,
        "vectorWeight": 0.7,    // 70% 语义相似度
        "textWeight": 0.3,      // 30% 关键词匹配
        "candidateMultiplier": 4, // 扩大搜索池（关键！）
        "mmr": {                // 最大边际相关性
          "enabled": true,
          "lambda": 0.7         // 多样性权重
        },
        "temporalDecay": {      // 时间衰减
          "enabled": true,
          "halfLifeDays": 30    // 30 天半衰期
        }
      }
    }
  }
}
```

**关键参数说明**：
- `candidateMultiplier`: 设置为 4 可显著提升结果质量
- `mmr.lambda`: 0.7 平衡相关性和多样性
- `halfLifeDays`: 30 天让新记忆优先级更高

### 3. Embedding Provider 选择

| Provider | 自动检测 | 成本 | 质量 | 推荐场景 |
|----------|---------|------|------|---------|
| `ollama` | ❌ | 免费 | ⭐⭐⭐ | 本地部署、隐私敏感 |
| `gemini` | ✅ | 免费额度足 | ⭐⭐⭐⭐ | 默认推荐 |
| `openai` | ✅ | $0.02/1K tokens | ⭐⭐⭐⭐⭐ | 生产环境 |
| `voyage` | ✅ | 中等 | ⭐⭐⭐⭐⭐ | 专业检索 |
| `mistral` | ✅ | 中等 | ⭐⭐⭐⭐ | 欧洲服务器 |

**推荐配置**：
```json
{
  "memorySearch": {
    "provider": "gemini",
    "model": "gemini-embedding-001"
  }
}
```

### 4. 三个关键实践（来自社区）

#### ✅ 实践 1：把持久化规则写入文件，而不是聊天

```markdown
# MEMORY.md

## 用户偏好
- 偏好 TypeScript 而非 JavaScript
- 喜欢简洁的代码风格
- 工作时间：9:00-18:00 (Asia/Shanghai)

## 重要规则
- 执行批量操作前必须获得明确批准
- 删除文件前必须确认
- 敏感信息必须加密存储
```

**为什么**：聊天中的指令在 compaction 后会丢失，文件中的规则永久保存。

#### ✅ 实践 2：确保 memory flush 已启用

检查配置：
```json
{
  "agents": {
    "defaults": {
      "contextPruning": {
        "mode": "cache-ttl",
        "ttl": "5m"
      }
    }
  }
}
```

**为什么**：OpenClaw 在 compaction 前有安全网，但需要足够缓冲触发。

#### ✅ 实践 3：强制检索

在 `AGENTS.md` 中添加：
```markdown
## 记忆使用规则

1. **搜索记忆后再行动** - 执行任何操作前必须调用 `memory_search`
2. **记录重要决策** - 每个重要决定必须写入 `MEMORY.md`
3. **每日总结** - 每天结束时总结关键信息到当日笔记
```

**为什么**：避免 AI 猜测而不是查阅笔记。

---

## 🔧 故障排查

### 问题 1：记忆搜索不工作

**症状**：`memory_search` 返回空结果

**排查步骤**：
```bash
# 1. 检查索引状态
openclaw memory status

# 2. 检查 provider 配置
openclaw config get agents.defaults.memorySearch

# 3. 手动重建索引
openclaw memory index --force

# 4. 查看日志
openclaw logs --follow --plain | grep memory
```

**常见原因**：
- ❌ 未配置 embedding provider
- ❌ 索引未建立
- ❌ 文件路径不正确

### 问题 2：搜索结果不相关

**症状**：`memory_search` 返回不相关内容

**解决方案**：
```json
{
  "memorySearch": {
    "query": {
      "hybrid": {
        "candidateMultiplier": 4,  // 扩大候选池
        "mmr": {
          "enabled": true,        // 启用多样性
          "lambda": 0.7
        }
      },
      "maxResults": 8              // 增加返回数量
    }
  }
}
```

### 问题 3：记忆在 compaction 后丢失

**症状**：长对话后 AI 忘记之前的指令

**根本原因**：指令只存在于聊天中，未写入文件

**解决方案**：
1. 将重要规则写入 `MEMORY.md` 或 `AGENTS.md`
2. 启用 memory flush
3. 使用 `memory_get` 主动读取记忆

---

## 📊 配置检查清单

### Memory Core 配置

- [ ] `memorySearch.enabled: true`
- [ ] `memorySearch.provider` 已配置（或使用自动检测）
- [ ] `sync.watch: true`（自动索引）
- [ ] `sync.watchDebounceMs: 1500`（防抖）
- [ ] `query.maxResults: 8`（返回数量）
- [ ] `query.hybrid.enabled: true`（混合搜索）

### QMD 额外配置

- [ ] QMD binary 已安装 (`npm install -g @tobilu/qmd`)
- [ ] QMD 在 PATH 中
- [ ] `memory.backend: "qmd"`
- [ ] `qmd.paths` 配置额外目录（如需要）
- [ ] `qmd.sessions.enabled: true`（如需要会话索引）

### Honcho 额外配置

- [ ] 插件已安装 (`openclaw plugins install @honcho-ai/openclaw-honcho`)
- [ ] Honcho setup 已完成
- [ ] API Key 已配置（或使用自托管）
- [ ] `workspaceId` 已设置

---

## 🎓 学习资源

### 官方文档
- [Memory Overview](https://docs.openclaw.ai/concepts/memory)
- [Builtin Memory Engine](https://docs.openclaw.ai/concepts/memory-builtin)
- [QMD Memory Engine](https://docs.openclaw.ai/concepts/memory-qmd)
- [Honcho Memory](https://docs.openclaw.ai/concepts/memory-honcho)
- [Memory Configuration Reference](https://docs.openclaw.ai/reference/memory-config)

### 社区资源
- [OpenClaw Memory Masterclass](https://velvetshark.com/openclaw-memory-masterclass)
- [2026 Complete Guide to OpenClaw memorySearch](https://dev.to/czmilo/2026-complete-guide-to-openclaw-memorysearch-supercharge-your-ai-assistant-49oc)
- [Tested every OpenClaw memory plugin](https://www.reddit.com/r/openclaw/comments/1s2574y/i_tested_every_openclaw_memory_plugin_so_you_dont/)

### GitHub Repos
- [openclaw/openclaw](https://github.com/openclaw/openclaw) - 主仓库
- [tobi/qmd](https://github.com/tobi/qmd) - QMD 搜索引擎
- [plastic-labs/openclaw-honcho](https://github.com/plastic-labs/openclaw-honcho) - Honcho 插件
- [CortexReach/memory-lancedb-pro](https://github.com/CortexReach/memory-lancedb-pro) - LanceDB Pro 插件

---

## 📝 版本历史

| 版本 | 日期 | 更新内容 |
|------|------|---------|
| 1.0 | 2026-04-09 | 初始版本，覆盖 4 种插件 |

---

*本文档由 AI-Note 社区维护，遵循 MIT 许可证。*
