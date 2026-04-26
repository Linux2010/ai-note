# LangGraph 状态持久化与检查点系统

## 1. 为什么需要持久化

没有持久化的 Agent 图:
- 每次执行都是全新的，丢失上下文
- 崩溃后无法恢复
- 无法支持对话历史
- 无法实现"人类在环"
- 无法调试和回看

持久化的核心价值:
- **对话记忆**: 跨多次调用的状态连续
- **崩溃恢复**: 从上次检查点恢复执行
- **人类在环**: 暂停、查看、编辑、继续
- **调试能力**: 回看每一步的状态变更
- **时间旅行**: 从任意历史状态重新执行

## 2. Checkpointer 架构

### 2.1 核心组件

```
┌───────────────────────────────────────────────────┐
│                  Graph.execute()                   │
│                         │                          │
│              ┌──────────▼──────────┐               │
│              │   Checkpointer      │               │
│              │  ┌───────────────┐  │               │
│              │  │ writes:       │  │               │
│              │  │ - thread_id   │  │               │
│              │  │ - checkpoint  │  │               │
│              │  │ - metadata    │  │               │
│              │  │ - parent_ts   │  │               │
│              │  └───────────────┘  │               │
│              │                     │               │
│              │  ┌───────────────┐  │               │
│              │  │ reads:        │  │               │
│              │  │ - get_latest  │  │               │
│              │  │ - get_by_ts   │  │               │
│              │  │ - list_all    │  │               │
│              │  └───────────────┘  │               │
│              └──────────┬──────────┘               │
│                         │                          │
│              ┌──────────▼──────────┐               │
│              │  Storage Backend    │               │
│              │  (Memory/Redis/...) │               │
│              └─────────────────────┘               │
└───────────────────────────────────────────────────┘
```

### 2.2 Checkpoint 数据结构

```python
@dataclass
class Checkpoint:
    """一个检查点包含以下信息"""
    v: int                            # 版本
    id: str                           # 检查点 ID (时间戳)
    ts: str                           # 时间戳字符串
    pending_sends: list               # 待发送的消息 (用于并行)
    channel_values: dict              # State 的完整快照
    metadata: dict                    # 用户自定义元数据
    parent_config: tuple              # 父检查点的配置 (用于版本树)

@dataclass
class CheckpointMetadata:
    """检查点的元数据"""
    source: str           # "input" | "update"
    step: int             # 在第几步创建的
    writes: dict | None   # 哪个节点写了什么
    parent_id: str | None # 父检查点的 ID (用于回溯)
```

### 2.3 执行过程中的检查点时机

```
invoke(input)
  │
  ├─ Step 0: 保存 START 前的 checkpoint (parent_id=None)
  │
  ├─ Step 1: 执行 Node A
  │  └─ 保存 checkpoint (step=1, 写入 Node A 的返回)
  │
  ├─ Step 2: 执行 Node B
  │  └─ 保存 checkpoint (step=2, 写入 Node B 的返回)
  │
  ├─ Step 3: 执行 Node C
  │  └─ 保存 checkpoint (step=3, 写入 Node C 的返回)
  │
  └─ END
```

每个 checkpoint 都是 State 的一个 **完整快照**，可以通过时间戳精确恢复到任意一步。

## 3. 内置 Checkpointer 实现

### 3.1 MemorySaver (内存)

```python
from langgraph.checkpoint.memory import MemorySaver

# 最简单的 checkpointer, 数据存在 Python dict 中
checkpointer = MemorySaver()

graph = builder.compile(checkpointer=checkpointer)

# 执行时需要传入 thread_id
config = {"configurable": {"thread_id": "user-123"}}
result = graph.invoke(
    {"messages": [("user", "你好")]},
    config=config
)

# 同一 thread 的后续调用会恢复之前的状态
result2 = graph.invoke(
    {"messages": [("user", "还记得我刚才说什么吗?")]},
    config=config  # 相同的 thread_id
)
```

特点:
- 零配置，适合开发调试
- 进程重启后数据丢失
- 线程安全
- 不支持跨进程共享

### 3.2 SqliteSaver (SQLite)

```python
from langgraph.checkpoint.sqlite import SqliteSaver

# 使用 SQLite 持久化
with SqliteSaver.from_conn_string("/tmp/checkpoints.db") as checkpointer:
    graph = builder.compile(checkpointer=checkpointer)

    config = {"configurable": {"thread_id": "session-1"}}
    graph.invoke({"messages": [("user", "Hello")]}, config=config)

    # 数据持久化到磁盘，重启后仍然存在
```

特点:
- 轻量级，无需额外服务
- 单进程写入 (SQLite 限制)
- 适合个人应用和原型开发
- 生产环境不推荐

### 3.3 PostgresSaver (PostgreSQL)

```python
from langgraph.checkpoint.postgres import PostgresSaver
from psycopg.conn import Connection

DB_URL = "postgresql://user:pass@localhost:5432/langgraph"

# 方式一: 连接字符串
with PostgresSaver.from_conn_string(DB_URL) as checkpointer:
    # 自动创建表
    checkpointer.setup()
    graph = builder.compile(checkpointer=checkpointer)

# 方式二: 连接池 (生产推荐)
from psycopg_pool import ConnectionPool

pool = ConnectionPool(conninfo=DB_URL, max_size=20)
checkpointer = PostgresSaver(pool)
graph = builder.compile(checkpointer=checkpointer)
```

特点:
- 生产级可靠性
- 支持并发写入
- ACID 事务保证
- 支持查询/过滤
- 推荐生产环境使用

### 3.4 RedisSaver (Redis)

```python
from langgraph.checkpoint.redis import RedisSaver

checkpointer = RedisSaver.from_conn_url("redis://localhost:6379")
graph = builder.compile(checkpointer=checkpointer)
```

特点:
- 高性能 (内存数据库)
- 支持持久化 (AOF/RDB)
- 适合高并发场景
- 需要 Redis 服务

## 4. Thread 管理

### 4.1 Thread 的概念

```
Thread 是检查点的逻辑容器。
每个 thread_id 对应一个独立的对话/会话。

Thread A:  checkpoint_1 → checkpoint_2 → checkpoint_3
Thread B:  checkpoint_1 → checkpoint_2
Thread C:  checkpoint_1
```

### 4.2 多 Thread 操作

```python
# 不同用户/会话使用不同 thread_id
def handle_user_message(user_id: str, message: str):
    config = {"configurable": {"thread_id": user_id}}

    # 第一次调用: 创建新 thread
    if not is_first_message(user_id):
        result = graph.invoke(
            {"messages": [("user", message)]},
            config=config
        )
    return result["messages"][-1].content

# 查询某个 thread 的所有检查点
from langgraph.checkpoint.base import Checkpoint

all_checkpoints = checkpointer.list(config)
for c in all_checkpoints:
    print(f"Step {c.metadata['step']}: {c.ts}")
```

### 4.3 Thread 的生命周期

```python
# 1. 创建 Thread (隐式, 首次 invoke 时)
config = {"configurable": {"thread_id": "new-session-42"}}
graph.invoke(input, config=config)

# 2. 继续 Thread (恢复最新检查点)
graph.invoke(new_input, config=config)

# 3. 查看 Thread 状态
state = graph.get_state(config)
print(state.values)  # 当前 State 快照
print(state.next)    # 下一步要执行的节点

# 4. 查看所有检查点
history = list(checkpointer.list(config))

# 5. 从历史检查点恢复 (时间旅行)
old_config = history[0].config  # 回到第一步
graph.invoke(fix_input, config=old_config)  # 从那里继续
```

## 5. 时间旅行 (Time Travel)

### 5.1 基本概念

```
时间旅行 = 从任意历史检查点继续执行

History:
  CP1 ─→ CP2 ─→ CP3 ─→ CP4 ─→ CP5 (当前)
                    │
                    └─→ CP3b ─→ CP4b (分叉!)

用途:
- 调试: 回到 bug 前一步重新执行
- 人类在环: 编辑历史状态后继续
- A/B 测试: 从同一状态尝试不同路径
```

### 5.2 实际操作

```python
# 查看历史
history = list(checkpointer.list(config))

# 从指定检查点恢复并继续
old_state = graph.get_state(history[2].config)
print(f"State at checkpoint 3: {old_state.values}")

# 从历史点继续执行 (自动创建分叉)
graph.invoke(
    {"messages": [("user", "换个方式回答")]},
    config=history[2].config  # 从 checkpoint 3 继续
)
# 这会创建一个新的分叉，不影响主线
```

## 6. Store (跨线程持久化)

Store 与 Checkpointer 的区别:

```
Checkpointer:  每个 Thread 独立的数据 (对话历史)
Store:         所有 Thread 共享的数据 (用户偏好, 知识)

┌─────────────────────────────────────────────┐
│                 Store                       │
│  ┌─────────────┬──────────────────────────┐ │
│  │ Namespace   │   Data                   │ │
│  ├─────────────┼──────────────────────────┤ │
│  │ ["users",   │ { "theme": "dark",       │ │
│  │  "u123",    │   "prefs": { ... }}     │ │
│  │  "prefs"]   │                          │ │
│  ├─────────────┼──────────────────────────┤ │
│  │ ["tools",   │ { "last_used": ...,      │ │
│  │  "search",  │   "count": 42}          │ │
│  │  "stats"]   │                          │ │
│  └─────────────┴──────────────────────────┘ │
└─────────────────────────────────────────────┘
```

```python
from langgraph.store.memory import InMemoryStore

# 创建 Store
store = InMemoryStore()

# 写入数据 (按 namespace 组织)
store.put(["users", "u123", "prefs"], "settings",
          {"theme": "dark", "language": "zh"})

# 读取数据
item = store.get(["users", "u123", "prefs"], "settings")

# 搜索
results = store.search(["users"], filter={"theme": "dark"})

# 在节点中使用
def personal_agent(state: AgentState, store: BaseStore) -> dict:
    user_id = state["user_id"]
    prefs = store.get(["users", user_id, "prefs"], "settings")
    # 根据用户偏好生成个性化响应
    ...

graph = builder.compile(checkpointer=checkpointer, store=store)
```

## 7. 生产环境选择建议

| 场景 | 推荐 Checkpointer | 推荐 Store |
|------|-------------------|------------|
| 本地开发 | MemorySaver | InMemoryStore |
| 个人应用 | SqliteSaver | InMemoryStore |
| 单节点生产 | PostgresSaver | PostgresStore |
| 高并发集群 | PostgresSaver + Redis | Redis Store |
| 超大规模 | PostgresSaver + 分片 | 分布式 KV |
