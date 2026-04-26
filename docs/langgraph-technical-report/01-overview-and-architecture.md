# LangGraph 技术概述与架构分析

## 1. 什么是 LangGraph

LangGraph 是由 LangChain 团队开发的 **图结构化 Agent 框架**。它将 LLM 驱动的 Agent 工作流建模为有状态的计算图，提供细粒度的控制流、状态管理、持久化和人类在环等能力。

核心定位:
- **Agent  orchestration 框架** (不是 LLM 调用库)
- 构建在 LangChain 生态之上 (可独立使用)
- 同时支持 Python 和 TypeScript

关键理念: **Agent 是有状态的循环图，不是线性链**

## 2. 核心特性

### 2.1 有环图控制流

```
传统线性链:                          LangGraph:

Input → Chain → Output              Input → Node A ──→ Node B
                                                         │
                                                    ┌────┤
                                                    │    ▼
                                                    │  Node C
                                                    └────┘
                                                      (循环)
```

LangGraph 的图可以是任意的有向图，支持:
- **循环**: Agent 可以自我循环 (如 ReAct 模式)
- **分支**: 条件路由到不同节点
- **并行**: 多个节点同时执行
- **嵌套**: 图可以包含子图

### 2.2 细粒度状态控制

```
每个节点返回 State 的部分更新 → 框架自动合并

Node A 返回 { x: 1, y: 2 }
Node B 返回 { y: 3, z: 4 }
Node C 返回 { x: 5 }

最终 State: { x: 5, y: 3, z: 4 }
```

通过 Reducer 机制，每个字段可以定义不同的合并策略:
- `add_messages`: 消息列表合并 (按 ID 去重)
- `operator.add`: 列表追加
- 自定义 reducer: 完全控制合并逻辑

### 2.3 内置持久化

```
每次节点执行 → 自动保存检查点 (Checkpoint)

Thread: session-1
  ├── CP1 (执行 Node A 前)
  ├── CP2 (执行 Node A 后)
  ├── CP3 (执行 Node B 后)
  └── CP4 (执行 Node C 后)
```

支持:
- 内存 (MemorySaver)
- SQLite (SqliteSaver)
- PostgreSQL (PostgresSaver)
- Redis (RedisSaver)

### 2.4 流式输出

```
stream_mode 选择:
├── "values"   — 每步返回完整 State
├── "updates"  — 每步返回增量更新
├── "messages" — Token 级流式输出
├── "events"   — 自定义事件流
└── "custom"   — 手动写入
```

### 2.5 人类在环

```
Agent 执行 → interrupt() → 暂停等待人类
                          ↓
                    人类查看 / 编辑 / 批准
                          ↓
                    Agent 恢复执行
```

## 3. 整体架构

```
┌─────────────────────────────────────────────────────────┐
│                   Application Layer                      │
│  (你的 Agent 逻辑: 节点函数 + 图定义)                     │
└───────────────────────────┬─────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────┐
│                     LangGraph Core                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │
│  │ StateGraph  │  │  Node       │  │  Edge Router    │  │
│  │  (图构建)   │  │  (节点函数)  │  │  (路由决策)     │  │
│  └──────┬──────┘  └──────┬──────┘  └────────┬────────┘  │
│         └────────────────┼──────────────────┘           │
│                    ┌─────▼─────┐                        │
│                    │ Pregel    │  (图执行引擎)           │
│                    │  Engine   │                        │
│                    └─────┬─────┘                        │
└──────────────────────────┼──────────────────────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Checkpointer │  │   Store      │  │   Stream     │
│ (持久化)     │  │ (跨线程存储)  │  │ (流式协议)   │
│              │  │              │  │              │
│ MemorySaver  │  │ InMemory     │  │ 5 modes      │
│ Postgres     │  │ Postgres     │  │ events/values│
│ Redis        │  │ Redis        │  │ / messages   │
└──────────────┘  └──────────────┘  └──────────────┘
```

## 4. Pregel 执行引擎

LangGraph 的底层执行引擎基于 Google 的 **Pregel** 图计算模型:

```
Pregel 核心思想:
- 图被分解为顶点和边
- 每个顶点独立计算
- 消息沿边传递
- 同步屏障确保所有顶点完成后再进入下一轮

LangGraph 中的映射:
- 顶点 = Node (节点函数)
- 边 = Edge (控制流)
- 消息 = State 更新
- 轮次 = 每个节点执行一步
```

### 4.1 执行流程

```
invoke(input)
  │
  ├─ 1. 初始化 State (input → State)
  │
  ├─ 2. 找到所有可执行的节点 (无未满足依赖)
  │
  ├─ 3. 并行执行可执行节点
  │   ├─ Node A 执行 → 返回 State 更新
  │   └─ Node B 执行 → 返回 State 更新
  │
  ├─ 4. 合并 State 更新 (按 Reducer 规则)
  │
  ├─ 5. 保存检查点
  │
  ├─ 6. 找到下一轮可执行节点
  │
  ├─ 7. 重复步骤 3-6...
  │
  └─ 8. 到达 END → 返回最终 State
```

## 5. 与 LangChain 的关系

```
LangChain 生态:

┌─────────────────────────────────────────────┐
│           LangChain (总框架)                 │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐ │
│  │ LCEL     │  │ LangSmith│  │ LangGraph │ │
│  │ (链式)   │  │ (可观测) │  │ (图式)    │ │
│  │          │  │          │  │           │ │
│  │ A → B → C│  │ 追踪/调试│  │ 有环图    │ │
│  │ 线性流程  │  │ 评估     │  │ Agent 编排 │ │
│  └──────────┘  └──────────┘  └───────────┘ │
│                                             │
│  ┌──────────┐  ┌──────────┐                │
│  │ Tools    │  │ Models   │                │
│  │ (工具)   │  │ (模型)   │                │
│  └──────────┘  └──────────┘                │
└─────────────────────────────────────────────┘
```

LCEL vs LangGraph:
```
LCEL (LangChain Expression Language):
- 线性管道
- A → B → C
- 适合: 简单链式流程
- 优点: 简洁
- 缺点: 不支持循环/分支

LangGraph:
- 图结构
- 任意控制流
- 适合: 复杂 Agent
- 优点: 灵活
- 缺点: 学习曲线
```

## 6. 安装与快速开始

```bash
# Python
pip install langgraph

# TypeScript
npm install @langchain/langgraph
```

```python
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver
from typing import TypedDict, Annotated
from langgraph.graph.message import add_messages

# 1. 定义状态
class State(TypedDict):
    messages: Annotated[list, add_messages]

# 2. 定义节点
def chatbot(state: State):
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

# 3. 构建图
builder = StateGraph(State)
builder.add_node("chatbot", chatbot)
builder.add_edge(START, "chatbot")
builder.add_edge("chatbot", END)

# 4. 编译
graph = builder.compile()

# 5. 执行
result = graph.invoke({
    "messages": [("user", "你好")]
})
```
