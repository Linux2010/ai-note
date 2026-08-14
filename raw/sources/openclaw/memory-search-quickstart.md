# OpenClaw Memory Search 快速入门指南

> **版本**: 1.0  
> **最后更新**: 2026-04-09  
> **适用版本**: OpenClaw 2026.4.2+  
> **难度**: ⭐ 入门级  
> **阅读时间**: 10 分钟

---

## 📖 什么是 Memory Search？

**Memory Search** 是 OpenClaw 的语义记忆搜索功能，让你的 AI 助手能够：

- 🔍 **理解语义** - 搜索"投资目标"也能找到"财务规划"相关内容
- 🧠 **长期记忆** - 记住你的偏好、决策、重要信息
- 📝 **自动索引** - 自动扫描 Markdown 文件并建立索引
- ⚡ **快速检索** - 毫秒级返回相关记忆片段

### 工作原理

```
你的记忆文件 (Markdown)
    ↓
自动索引 (每 1.5 秒)
    ↓
SQLite 数据库 (向量 + 全文索引)
    ↓
混合搜索 (向量相似度 + 关键词匹配)
    ↓
返回最相关的记忆片段
```

---

## 🚀 5 分钟快速开始

### 步骤 1: 检查当前状态

```bash
# 检查 memory search 是否启用
openclaw memory status
```

**期望输出**:
```
Memory Search: enabled
Provider: ollama (or openai/gemini/etc)
Index: ~/.openclaw/memory/main.sqlite
Files indexed: MEMORY.md, memory/*.md
```

### 步骤 2: 创建记忆文件

```bash
# 创建记忆目录
mkdir -p ~/.openclaw/agents/main/workspace/memory

# 创建今日笔记
cat > ~/.openclaw/agents/main/workspace/memory/$(date +%Y-%m-%d).md << 'EOF'
# Session: $(date +%Y-%m-%d)

## 用户偏好
- 编程语言：偏好 TypeScript
- 代码风格：简洁、函数式
- 工作时间：9:00-18:00 (Asia/Shanghai)

## 重要决策
- 选择 PostgreSQL 而非 MongoDB（原因：事务支持更好）
- 使用 Docker 部署所有服务

## 项目信息
- 项目名称：Hermes AI Assistant
- 技术栈：Node.js + TypeScript + Docker
EOF
```

### 步骤 3: 配置 Memory Search

编辑 `~/.openclaw/openclaw.json`:

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
        }
      }
    }
  }
}
```

### 步骤 4: 重启 Gateway

```bash
openclaw gateway restart
```

### 步骤 5: 测试搜索

在 Telegram 或其他聊天界面中问：

```
我偏好什么编程语言？
```

**期望回答**:
```
根据你的记忆，你偏好 TypeScript，喜欢简洁、函数式的代码风格。
```

---

## ⚙️ 配置选项详解

### 基础配置

```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "enabled": true,              // 是否启用
        "provider": "ollama",         // embedding provider
        "sync": {
          "watch": true,              // 自动监控文件变化
          "watchDebounceMs": 1500     // 防抖时间 (毫秒)
        },
        "query": {
          "maxResults": 8,            // 最大返回结果数
          "hybrid": {
            "enabled": true,          // 启用混合搜索
            "vectorWeight": 0.7,      // 向量权重 (0-1)
            "textWeight": 0.3         // 关键词权重 (0-1)
          }
        }
      }
    }
  }
}
```

### Provider 选择

| Provider | 配置 | 成本 | 质量 | 推荐场景 |
|----------|------|------|------|---------|
| **Ollama** | `"provider": "ollama"` | 免费 | ⭐⭐⭐ | 本地部署、隐私敏感 |
| **Gemini** | `"provider": "gemini"` | 免费额度足 | ⭐⭐⭐⭐ | 默认推荐 |
| **OpenAI** | `"provider": "openai"` | $0.02/1K tokens | ⭐⭐⭐⭐⭐ | 生产环境 |
| **Voyage** | `"provider": "voyage"` | 中等 | ⭐⭐⭐⭐⭐ | 专业检索 |

**推荐配置** (本地):
```json
{
  "memorySearch": {
    "provider": "ollama",
    "model": "nomic-embed-text"
  }
}
```

**推荐配置** (云端):
```json
{
  "memorySearch": {
    "provider": "gemini",
    "model": "gemini-embedding-001"
  }
}
```

### 高级配置

```json
{
  "memorySearch": {
    "query": {
      "hybrid": {
        "enabled": true,
        "vectorWeight": 0.7,
        "textWeight": 0.3,
        "candidateMultiplier": 4,    // 扩大候选池 (关键！)
        "mmr": {                     // 最大边际相关性
          "enabled": true,
          "lambda": 0.7              // 多样性权重
        },
        "temporalDecay": {           // 时间衰减
          "enabled": true,
          "halfLifeDays": 30         // 30 天半衰期
        }
      }
    }
  }
}
```

---

## 📁 记忆文件组织

### 推荐结构

```
workspace/
├── MEMORY.md                    # 长期记忆（持久化事实）
├── memory/
│   ├── YYYY-MM-DD.md           # 每日笔记（自动加载今天 + 昨天）
│   ├── projects/               # 项目相关记忆
│   │   ├── project-a.md
│   │   └── project-b.md
│   ├── knowledge/              # 知识点整理
│   │   ├── meetings.md
│   │   └── decisions.md
│   └── reference/              # 参考资料
│       ├── contacts.md
│       └── passwords.md
└── DREAMS.md                   # （可选）梦境日记
```

### MEMORY.md 示例

```markdown
# 长期记忆

## 用户信息
- 姓名：张三
- 时区：Asia/Shanghai
- 语言：中文

## 偏好设置
- 编程语言：TypeScript > JavaScript
- 代码风格：简洁、函数式
- 文档风格：Markdown + 示例代码

## 技术栈
- 前端：React + TypeScript
- 后端：Node.js + PostgreSQL
- 部署：Docker + Kubernetes

## 重要规则
1. 执行批量操作前必须获得明确批准
2. 删除文件前必须确认
3. 敏感信息必须加密存储
```

### 每日笔记示例

```markdown
# Session: 2026-04-09

## 完成的任务
- [x] 配置 Memory Search
- [x] 创建记忆文件结构
- [x] 测试语义搜索

## 重要决策
- 选择 Ollama 作为 embedding provider（本地运行，隐私好）
- 使用混合搜索（向量 + 关键词）

## 学到的东西
- Memory Search 支持混合搜索
- 防抖时间设置为 1500ms 避免频繁索引
- candidateMultiplier 设置为 4 提升搜索质量

## 明日计划
- 优化搜索配置
- 添加更多记忆文件
- 测试跨会话记忆
```

---

## 🔧 常用命令

### 检查状态

```bash
# 查看 memory search 状态
openclaw memory status

# 查看索引文件
ls -lh ~/.openclaw/memory/*.sqlite

# 查看已索引文件
openclaw memory index --list
```

### 管理索引

```bash
# 手动重建索引
openclaw memory index --force

# 添加额外路径
openclaw memory index --add ~/notes/*.md

# 清除索引缓存
openclaw memory index --clear-cache
```

### 测试搜索

```bash
# 命令行搜索
openclaw memory search "投资目标"

# 带参数搜索
openclaw memory search "TypeScript" --limit 5 --verbose
```

---

## 💡 使用技巧

### 1. 使用自然语言查询

✅ **好**:
```
"我上次提到的项目是什么？"
"为什么选择 PostgreSQL？"
"我的工作时间是什么？"
```

❌ **差**:
```
"项目"
"PostgreSQL"
"时间"
```

### 2. 添加上下文

✅ **好**:
```
"关于 Hermes 项目的技术栈决策"
"上周讨论的数据库选择原因"
```

❌ **差**:
```
"技术栈"
"数据库"
```

### 3. 使用引用

在对话中引用记忆：

```
根据我的记忆，我说过...
我记得之前决定...
上次我们讨论过...
```

### 4. 定期整理

每周整理一次记忆文件：

```bash
# 合并碎片笔记
cat memory/2026-04-*.md >> MEMORY.md

# 清理旧文件
rm memory/2026-03-*.md

# 重建索引
openclaw memory index --force
```

---

## 🐛 常见问题

### Q1: 搜索返回空结果

**症状**: `memory_search` 返回空

**排查**:
```bash
# 1. 检查是否启用
openclaw memory status

# 2. 检查 provider
openclaw config get agents.defaults.memorySearch

# 3. 重建索引
openclaw memory index --force

# 4. 查看日志
openclaw logs --follow | grep memory
```

**常见原因**:
- ❌ 未配置 embedding provider
- ❌ 索引未建立
- ❌ 文件路径不正确

### Q2: 搜索结果不相关

**解决方案**:
```json
{
  "memorySearch": {
    "query": {
      "hybrid": {
        "candidateMultiplier": 4,  // 扩大候选池
        "mmr": {
          "enabled": true,
          "lambda": 0.7
        }
      }
    }
  }
}
```

### Q3: 记忆在 compaction 后丢失

**原因**: 指令只存在于聊天中，未写入文件

**解决**:
1. 将重要规则写入 `MEMORY.md` 或 `AGENTS.md`
2. 启用 memory flush
3. 使用 `memory_get` 主动读取记忆

### Q4: 索引更新慢

**优化**:
```json
{
  "sync": {
    "watchDebounceMs": 3000  // 增加到 3 秒
  }
}
```

---

## 📚 进阶阅读

- [Memory Plugins Complete Guide](./memory-plugins-complete-guide.md) - 完整插件对比
- [Memory Search Best Practices](./memory-search-best-practices.md) - 最佳实践
- [LanceDB Comparison](./memory-lancedb-comparison.md) - LanceDB 深度分析

---

## 🎯 下一步

1. ✅ 完成快速开始配置
2. 📝 创建你的第一个记忆文件
3. 🔍 测试语义搜索
4. ⚙️ 调整配置参数
5. 📊 查看最佳实践文档

---

*本文档是 OpenClaw Memory Search 入门指南，适合新手快速上手。*

*更多高级用法请参考 [Memory Search Best Practices](./memory-search-best-practices.md)*
