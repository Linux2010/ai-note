# Elite Longterm Memory 完整实现指南

> 托管式、自迭代的 AI Agent 长期记忆系统

---

## 目录

1. [系统概述](#系统概述)
2. [架构设计](#架构设计)
3. [环境准备](#环境准备)
4. [L1-L4 实现](#l1-l4-实现)
5. [L6 Mem0 本地部署](#l6-mem0-本地部署)
6. [Cron 定时维护](#cron-定时维护)
7. [自我改进循环](#自我改进循环)
8. [使用指南](#使用指南)
9. [维护手册](#维护手册)

---

## 系统概述

### 目标

构建一个**托管式、自迭代**的长期记忆系统：

- ✅ 自动识别重要信息
- ✅ 自动存储和整理
- ✅ 跨会话自动工作
- ✅ 自我学习和改进

### 技术栈

| 组件 | 技术 | 说明 |
|------|------|------|
| LLM | qwen3.5-max (云端) | 事实提取、对话理解 |
| Embedding | Ollama + nomic-embed-text | 本地向量化 |
| 向量数据库 | LanceDB (OpenClaw 内置) | 语义搜索 |
| 自动提取 | Mem0 OSS | 自动事实识别 |
| 定时维护 | OpenClaw Cron | 记忆整理优化 |

---

## 架构设计

### 6层记忆架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    ELITE LONGTERM MEMORY                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   HOT RAM   │  │  WARM STORE │  │  COLD STORE │             │
│  │ SESSION-    │  │  LanceDB    │  │  Git-Notes  │             │
│  │ STATE.md    │  │  Vectors    │  │  Knowledge  │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│         │                │                │                     │
│         └────────────────┼────────────────┘                     │
│                          ▼                                      │
│                  ┌─────────────┐                                │
│                  │  MEMORY.md  │  ← Curated long-term           │
│                  │  + memory/  │    (human-readable)            │
│                  └─────────────┘                                │
│                          │                                      │
│                          ▼                                      │
│                  ┌─────────────┐                                │
│                  │    Mem0     │  ← Auto-extraction             │
│                  │  (Local)    │    (自动提取事实)               │
│                  └─────────────┘                                │
└─────────────────────────────────────────────────────────────────┘
```

### 层级说明

| 层级 | 名称 | 存储 | 自动化程度 | 作用 |
|------|------|------|------------|------|
| **L1** | HOT RAM | SESSION-STATE.md | ✅ WAL 自动 | 当前会话上下文，存活于 compaction |
| **L2** | WARM STORE | LanceDB | ✅ 自动召回 | 语义搜索，基于向量相似度 |
| **L3** | COLD STORE | Git-Notes | ⚠️ 半自动 | 分支感知的永久决策存储 |
| **L4** | CURATED | MEMORY.md + memory/ | ✅ 自动更新 | 人类可读的提炼智慧 |
| **L6** | AUTO-EXTRACTION | Mem0 | ✅ 完全自动 | 从对话自动提取事实 |

### 数据流

```
用户对话
    ↓
[Mem0 自动提取] ←── 新增：自动识别重要信息
    ↓
重要事实/偏好/决策
    ↓
[nomic-embed-text] ← 本地 Embedding
    ↓
向量 (768维)
    ↓
[LanceDB] ← OpenClaw 内置向量存储
    ↓
下次对话自动召回
```

---

## 环境准备

### 前置条件

- macOS / Linux
- Python 3.10+
- Node.js 18+
- Docker (可选，用于 Qdrant)
- OpenClaw 已安装

### 检查现有环境

```bash
# 检查 Python
python3 --version  # 需要 3.10+

# 检查 Node.js
node --version     # 需要 18+

# 检查 Ollama
ollama list        # 查看已安装模型

# 检查 OpenClaw
openclaw status
```

### 安装必要组件

```bash
# 1. 安装 nomic-embed-text (Embedding 模型)
ollama pull nomic-embed-text

# 2. 安装 Mem0
pip install mem0ai

# 3. (可选) 启动 Qdrant 向量数据库
docker run -d -p 6333:6333 qdrant/qdrant
```

---

## L1-L4 实现

### L1: HOT RAM (SESSION-STATE.md)

**文件位置**: `~/.openclaw/agents/coding/workspace/SESSION-STATE.md`

```markdown
# SESSION-STATE.md — Active Working Memory

## Current Task
[当前正在进行的任务]

## Key Context
- 用户偏好: ...
- 重要决策: ...
- 当前阻塞: ...

## Pending Actions
- [ ] 待办事项

## Recent Decisions
- 2026-03-06: 决策内容

---
*Last updated: [时间戳]*
```

**WAL 协议**：先写后响应，确保上下文不丢失

### L2: WARM STORE (LanceDB)

**配置位置**: `~/.openclaw/openclaw.json`

```json
{
  "memorySearch": {
    "enabled": true,
    "provider": "ollama"
  }
}
```

**存储位置**: `~/.openclaw/memory/*.sqlite`

**使用方式**:
- 自动：每次对话自动搜索相关记忆
- 手动：`memory_search` 工具

### L3: COLD STORE (Git-Notes)

**初始化**:
```bash
cd ~/.openclaw/agents/coding/workspace
git init
```

**存储决策**:
```bash
# 存储决策到 Git-Notes
git notes add -m '{"type":"decision","content":"决策内容","date":"2026-03-06"}' HEAD

# 查看存储的决策
git notes show HEAD
```

**分支感知**：不同分支可以有独立的记忆

### L4: CURATED ARCHIVE (MEMORY.md + memory/)

**文件结构**:
```
workspace/
├── MEMORY.md              # 长期记忆（人类可读）
└── memory/
    ├── 2026-03-06.md      # 每日日志
    ├── 2026-03-05.md
    └── topics/            # 主题文件
```

**MEMORY.md 模板**:
```markdown
# Core Memory

## 用户偏好
- 偏好1
- 偏好2

## 重要决策
- 决策1: 原因和结果

## 经验教训
- 教训1: 如何避免

## 当前项目
- 项目状态
```

---

## L6 Mem0 本地部署

### 为什么需要 Mem0？

**问题**：L1-L4 需要用户明确说「记住」才能存储信息

**解决**：Mem0 自动从对话中提取事实，无需手动触发

### 架构对比

| 特性 | L1-L4 | L6 Mem0 |
|------|-------|---------|
| 触发方式 | 手动说「记住」 | 自动提取 |
| Token 消耗 | 高 (原始历史) | **低 80%** |
| 隐私 | 完全本地 | 可完全本地 |

### 配置文件

创建 `~/.openclaw/mem0-config.yaml`:

```yaml
# Mem0 本地配置
vector_store:
  provider: lancedb  # 使用 OpenClaw 内置的 LanceDB
  config:
    path: ~/.openclaw/memory/mem0

llm:
  provider: openai  # 使用云端 qwen3.5-max
  config:
    model: qwen3.5-max
    api_key: ${QWEN_API_KEY}
    base_url: https://dashscope.aliyuncs.com/compatible-mode/v1

embedder:
  provider: ollama  # 本地 Embedding
  config:
    model: nomic-embed-text
```

### 环境变量

```bash
# 添加到 ~/.zshrc
export QWEN_API_KEY="your-qwen-api-key"
```

### Python 集成脚本

创建 `~/.openclaw/scripts/mem0-client.py`:

```python
#!/usr/bin/env python3
"""
Mem0 本地客户端 - 自动事实提取
"""

from mem0 import Memory
import os

# 加载配置
config = {
    "vector_store": {
        "provider": "lancedb",
        "config": {
            "db_path": os.path.expanduser("~/.openclaw/memory/mem0")
        }
    },
    "llm": {
        "provider": "openai",
        "config": {
            "model": "qwen3.5-max",
            "api_key": os.environ.get("QWEN_API_KEY"),
            "base_url": "https://dashscope.aliyuncs.com/compatible-mode/v1"
        }
    },
    "embedder": {
        "provider": "ollama",
        "config": {
            "model": "nomic-embed-text"
        }
    }
}

memory = Memory.from_config(config)

def add_memory(messages, user_id="coding"):
    """添加记忆（自动提取事实）"""
    return memory.add(messages, user_id=user_id)

def search_memory(query, user_id="coding"):
    """搜索记忆"""
    return memory.search(query, user_id=user_id)

def get_all_memories(user_id="coding"):
    """获取所有记忆"""
    return memory.get_all(user_id=user_id)

# CLI 接口
if __name__ == "__main__":
    import sys
    
    if len(sys.argv) < 2:
        print("Usage: mem0-client.py [add|search|list] [args...]")
        sys.exit(1)
    
    command = sys.argv[1]
    
    if command == "add":
        text = " ".join(sys.argv[2:])
        result = add_memory([{"role": "user", "content": text}])
        print(f"✓ 记忆已添加: {result}")
    
    elif command == "search":
        query = " ".join(sys.argv[2:])
        results = search_memory(query)
        for r in results:
            print(f"- {r['memory']}")
    
    elif command == "list":
        memories = get_all_memories()
        for m in memories:
            print(f"- [{m['created_at']}] {m['memory']}")
```

---

## Cron 定时维护

### 维护任务

| 任务 | 频率 | 作用 |
|------|------|------|
| 记忆索引优化 | 每天 | 重建向量索引，提高搜索效率 |
| 过期记忆清理 | 每周 | 清理过期的临时记忆 |
| 记忆分层整理 | 每周 | 将重要记忆从 L1 移动到 L4 |
| 记忆备份 | 每月 | 本地备份记忆文件 |

### 配置 Cron

**方式1: OpenClaw Cron**

```bash
# 每周日凌晨 4 点执行记忆维护
openclaw cron add <<EOF
{
  "name": "memory-maintenance",
  "schedule": "0 4 * * 0",
  "action": "script",
  "script": "~/.openclaw/scripts/memory-maintenance.sh"
}
EOF
```

**方式2: 系统 Cron**

```bash
# 编辑 crontab
crontab -e

# 添加任务
0 4 * * 0 /bin/bash ~/.openclaw/scripts/memory-maintenance.sh >> ~/.openclaw/logs/maintenance.log 2>&1
```

### 维护脚本

创建 `~/.openclaw/scripts/memory-maintenance.sh`:

```bash
#!/bin/bash
# 记忆维护脚本

LOG_FILE=~/.openclaw/logs/maintenance.log
MEMORY_DIR=~/.openclaw/agents/coding/workspace/memory
DATE=$(date +%Y-%m-%d)

echo "[$DATE] 开始记忆维护..." >> $LOG_FILE

# 1. 检查记忆库大小
DB_SIZE=$(du -sh ~/.openclaw/memory/*.sqlite | awk '{sum+=$1} END {print sum}')
echo "记忆库大小: ${DB_SIZE}MB" >> $LOG_FILE

# 2. 清理超过 30 天的临时记忆
# (可根据需要实现)

# 3. 归档旧的每日日志
find $MEMORY_DIR -name "*.md" -mtime +30 -exec mv {} $MEMORY_DIR/archive/ \; 2>/dev/null

# 4. Git 提交记忆更新
cd ~/.openclaw/agents/coding/workspace
git add -A
git commit -m "chore: Memory maintenance - $DATE" 2>/dev/null

echo "[$DATE] 记忆维护完成" >> $LOG_FILE
```

---

## 自我改进循环

### 设计理念

```
错误发生 → 记录学习 → 更新记忆 → 下次避免
```

### 实现机制

**1. 错误捕获**

文件位置: `~/.openclaw/agents/coding/workspace/.learnings/`

```
.learnings/
├── ERRORS.md      # 错误记录
├── LEARNINGS.md   # 学习总结
└── FEATURE_REQUESTS.md  # 功能需求
```

**2. 自动学习流程**

```python
# 当发生错误时
def on_error(error_type, error_message, context):
    # 1. 记录到 ERRORS.md
    log_error(error_type, error_message, context)
    
    # 2. 生成学习点
    learning = generate_learning(error_type, error_message)
    
    # 3. 更新 LEARNINGS.md
    update_learnings(learning)
    
    # 4. 同步到 MEMORY.md
    update_memory(learning)
    
    # 5. 存入向量记忆
    memory_store(learning, category="lesson", importance=0.9)
```

**3. 主动避免机制**

```markdown
# 在 SESSION-STATE.md 中添加

## Lessons to Avoid
- 不要使用 rm -rf，使用 trash
- 提交前必须运行测试
- 环境变量不能硬编码
```

### Heartbeat 集成

**文件**: `~/.openclaw/agents/coding/workspace/HEARTBEAT.md`

```markdown
# Heartbeat 任务

## 每次检查
- [ ] 检查是否有新的错误记录
- [ ] 检查是否有待处理的学习点

## 每周任务
- [ ] 审查 ERRORS.md，总结规律
- [ ] 更新 MEMORY.md 中的经验教训
- [ ] 清理已解决的问题
```

---

## 使用指南

### 日常使用

**自动工作**（无需干预）:
- Mem0 自动从对话中提取事实
- L2 自动召回相关记忆
- WAL 协议自动保存上下文

**手动触发**:
```bash
# 存储重要决策
git notes add -m '{"type":"decision","content":"..."}' HEAD

# 搜索记忆
memory_search "项目名称"

# 查看所有记忆
cat MEMORY.md
cat memory/2026-03-06.md
```

### 记忆查询

**方式1: 对话中自然询问**
```
用户: "之前我们讨论的关于认证的方案是什么？"
AI: [自动搜索记忆并回答]
```

**方式2: 直接查看文件**
```bash
# 查看当前上下文
cat SESSION-STATE.md

# 查看长期记忆
cat MEMORY.md

# 查看每日日志
cat memory/2026-03-06.md
```

### 记忆更新

**自动更新**:
- Mem0 自动提取新事实
- WAL 协议自动更新 SESSION-STATE.md

**手动更新**:
```bash
# 编辑长期记忆
vim MEMORY.md

# 添加每日日志
vim memory/$(date +%Y-%m-%d).md

# 提交更改
git add -A && git commit -m "update: memory"
```

---

## 维护手册

### 记忆库位置

| 类型 | 位置 | 说明 |
|------|------|------|
| 向量记忆 | `~/.openclaw/memory/*.sqlite` | LanceDB 数据库 |
| 热记忆 | `SESSION-STATE.md` | 当前会话上下文 |
| 长期记忆 | `MEMORY.md` | 精炼的记忆 |
| 每日日志 | `memory/*.md` | 原始记录 |
| Git 记忆 | Git Notes | 分支感知决策 |

### 备份策略

```bash
# 手动备份
tar -czf ~/backup/openclaw-memory-$(date +%Y%m%d).tar.gz \
  ~/.openclaw/memory/ \
  ~/.openclaw/agents/coding/workspace/MEMORY.md \
  ~/.openclaw/agents/coding/workspace/memory/ \
  ~/.openclaw/agents/coding/workspace/SESSION-STATE.md

# 自动备份 (添加到 crontab)
0 0 1 * * tar -czf ~/backup/openclaw-memory-$(date +\%Y\%m\%d).tar.gz ~/.openclaw/memory/ ~/.openclaw/agents/coding/workspace/MEMORY.md ~/.openclaw/agents/coding/workspace/memory/
```

### 故障排除

**问题1: memory_search 返回空**

```bash
# 检查 embedding 服务
ollama list | grep nomic

# 检查向量数据库
ls -la ~/.openclaw/memory/*.sqlite

# 重启 gateway
openclaw gateway restart
```

**问题2: Mem0 无法启动**

```bash
# 检查依赖
pip install --upgrade mem0ai

# 检查配置
cat ~/.openclaw/mem0-config.yaml

# 检查 API Key
echo $QWEN_API_KEY
```

**问题3: 记忆丢失**

```bash
# 从 Git 恢复
cd ~/.openclaw/agents/coding/workspace
git log --oneline
git checkout HEAD~1 -- MEMORY.md

# 从备份恢复
tar -xzf ~/backup/openclaw-memory-YYYYMMDD.tar.gz -C /
```

---

## 总结

### 已实现功能

| 功能 | 状态 | 说明 |
|------|------|------|
| L1 HOT RAM | ✅ | WAL 协议，存活于 compaction |
| L2 WARM STORE | ✅ | LanceDB + Ollama 本地 embedding |
| L3 COLD STORE | ✅ | Git-Notes 分支感知 |
| L4 CURATED | ✅ | MEMORY.md + 每日日志 |
| L6 AUTO-EXTRACTION | 📋 | Mem0 本地部署（待实现） |
| Cron 维护 | 📋 | 定时任务（待配置） |
| 自我改进 | 📋 | 错误学习循环（待完善） |

### 下一步

1. **部署 Mem0 本地版** - 实现自动事实提取
2. **配置 Cron 任务** - 定期维护记忆
3. **完善自我改进机制** - 从错误中学习

---

*文档版本: 1.0.0*
*最后更新: 2026-03-06*