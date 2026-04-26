# LangGraph 核心概念详解

## 1. State（状态）

State 是 LangGraph 中所有数据的核心载体。每个图在执行时维护一个 State 对象，该对象在节点之间传递并被逐步修改。

### 1.1 StateSchema

```python
from typing import Annotated, TypedDict, Any
from typing_extensions import TypedDict
from langgraph.graph.message import add_messages
from langgraph.graph import StateGraph, END, START

# 方式一: TypedDict (推荐, 简单)
class AgentState(TypedDict):
    messages: Annotated[list, add_messages]  # 合并策略
    user_input: str
    result: str
    step_count: int

# 方式二: dataclass (支持更复杂类型)
from dataclasses import dataclass, field

@dataclass
class AgentState:
    messages: Annotated[list, add_messages]
    context: dict = field(default_factory=dict)
    metadata: dict = field(default_factory=dict)
```

### 1.2 Reducer（归约器）

Reducer 决定新值如何合并到现有 State 中。这是 LangGraph 状态管理的核心机制。

```python
# 内置 Reducer

# 1. add_messages — 消息合并 (按 ID 去重/追加)
from langgraph.graph.message import add_messages
messages: Annotated[list, add_messages]

# 2. operator.add — 列表追加
import operator
history: Annotated[list, operator.add]

# 3. 自定义 Reducer
def merge_dicts(current: dict, next: dict) -> dict:
    """用 next 更新 current（覆盖已存在的 key）"""
    current.update(next)
    return current

class AgentState(TypedDict):
    data: Annotated[dict, merge_dicts]

# 4. 最后写入者获胜（无 reducer 时的默认行为）
class SimpleState(TypedDict):
    count: int  # 直接覆盖
```

### 1.3 State 更新规则

```
┌─────────────┐     ┌─────────────┐
│  Node A     │────>│  Node B     │
│  returns:   │     │  returns:   │
│  {x: 1,     │     │  {y: 2,     │
│   z: 3}     │     │   x: None}  │
└─────────────┘     └─────────────┘
      │                   │
      ▼                   ▼
  State = {x: 1,      State = {x: None,
              z: 3}                y: 2,
                                  z: 3}

规则:
- Node 返回的 dict 会 merge 到当前 State
- 每个 key 的 merge 方式由 reducer 决定
- 如果返回 None, 不会修改 State
- 返回 {} 也是合法的 (不修改任何字段)
```

## 2. StateGraph（状态图）

StateGraph 是 LangGraph 的核心构建块，用于定义 Agent 的控制流。

### 2.1 图构建流程

```
创建 → 添加节点 → 添加边 → 编译 → 执行
 │        │          │        │       │
 │        │          │        │       └─ graph.invoke(input)
 │        │          │        └─ graph.compile(checkpointer=...)
 │        │          └─ add_edge / add_conditional_edges
 │        └─ add_node("name", function)
 └─ StateGraph(StateSchema)
```

### 2.2 完整构建示例

```python
from langgraph.graph import StateGraph, END, START

# 1. 创建图
builder = StateGraph(AgentState)

# 2. 添加节点 (函数)
def chatbot(state: AgentState) -> dict:
    """AI 助手节点"""
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

def tool_executor(state: AgentState) -> dict:
    """工具执行节点"""
    last_msg = state["messages"][-1]
    tool_call = last_msg.tool_calls[0]
    tool = tools_by_name[tool_call["name"]]
    result = tool.invoke(tool_call["args"])
    return {"messages": [result]}

# 3. 注册节点
builder.add_node("chatbot", chatbot)
builder.add_node("tools", tool_executor)

# 4. 添加边
# 固定边: START → chatbot
builder.add_edge(START, "chatbot")
# 固定边: chatbot → END (当不调用工具时)
builder.add_conditional_edges(
    "chatbot",
    should_continue,    # 路由函数
    {
        "continue": "tools",    # 条件 → 目标节点
        "end": END
    }
)
# 固定边: tools → chatbot (循环!)
builder.add_edge("tools", "chatbot")

# 5. 编译
graph = builder.compile()

# 6. 执行
result = graph.invoke({"messages": [("user", "今天天气怎样?")]})
```

## 3. 节点 (Nodes)

节点是图中执行具体逻辑的单元，本质上是接受 State 片段并返回 State 更新的 Python 可调用对象。

### 3.1 节点函数签名

```python
# 标准节点
def my_node(state: AgentState) -> dict:
    return {"key": "new_value"}

# 带配置的节点
def my_node_with_config(state: AgentState, config: RunnableConfig) -> dict:
    llm = get_llm(config["configurable"]["model"])
    ...

# 带 Store 的节点 (跨线程持久化)
def my_node_with_store(state: AgentState, store: BaseStore) -> dict:
    user_prefs = store.get(["prefs", user_id])
    ...
```

### 3.2 节点最佳实践

```python
# ✅ 推荐: 单一职责，返回值明确
def extract_keywords(state: AgentState) -> dict:
    text = state["user_input"]
    keywords = extract(text)
    return {"keywords": keywords}

# ❌ 避免: 在节点中修改传入的 state 对象 (应返回新 dict)
def bad_node(state: AgentState) -> dict:
    state["count"] += 1  # 不要直接修改输入
    return state

# ✅ 推荐: 返回 None 表示"不修改 State"
def logging_node(state: AgentState) -> None:
    print(state["messages"])  # 只读不写
    return None  # 或不写 return

# ✅ 推荐: 节点内处理异常，不要让异常泄漏到图调度
def robust_node(state: AgentState) -> dict:
    try:
        result = call_external_api(state["data"])
        return {"result": result}
    except Exception as e:
        return {"error": str(e), "status": "failed"}
```

### 3.3 节点参数注入

LangGraph 支持自动注入特殊参数到节点函数：

```python
# 框架自动注入的参数名 (无需调用方传):
# - config: RunnableConfig (当前执行配置)
# - store: BaseStore (跨线程持久化存储)
# - writer: StreamWriter (流式写入)

def my_node(state: AgentState, config: RunnableConfig, store: BaseStore):
    # config 和 store 由框架自动注入
    user_id = config["configurable"]["user_id"]
    prefs = store.get(["user", user_id, "prefs"])
    ...
```

## 4. 边 (Edges)

边定义了节点之间的控制流，决定执行顺序。

### 4.1 边类型

```
┌──────────┬──────────────┬──────────────────────────────┐
│ 类型     │ 函数         │ 用途                         │
├──────────┼──────────────┼──────────────────────────────┤
│ 固定边   │ add_edge     │ A → B (无条件跳转)            │
│ 条件边   │ add_cond_    │ A → [B, C, D] (运行时决策)    │
│          │ itional_edges│                              │
│ 入口     │ add_edge     │ START → A (图的起点)           │
│          │ (START, ...) │                              │
│ 出口     │ add_edge     │ A → END (图的终点)            │
│          │ (..., END)   │                              │
└──────────┴──────────────┴──────────────────────────────┘
```

### 4.2 固定边

```python
# A 执行完后一定执行 B
builder.add_edge("node_a", "node_b")
```

### 4.3 条件边

```python
# 路由函数: 返回目标节点名
def route_on_status(state: AgentState) -> str:
    if state["result"] == "success":
        return "handle_success"
    elif state["result"] == "failure":
        return "handle_failure"
    else:
        return "retry"

# 映射: 路由返回值 → 目标节点
builder.add_conditional_edges(
    "process",          # 源节点
    route_on_status,    # 路由函数 (返回字符串)
    {
        "handle_success": "success_handler",
        "handle_failure": "failure_handler",
        "retry": "process"  # 循环!
    }
)

# 简化写法: 直接返回目标节点名
def simple_route(state):
    if state["score"] > 80:
        return "pass_node"
    return "fail_node"

builder.add_conditional_edges("evaluator", simple_route)
```

### 4.4 并行边

```python
# A 执行完后，B 和 C 并行执行 (需 compile 时设置)
builder.add_edge("split", "process_left")
builder.add_edge("split", "process_right")
builder.add_edge("process_left", "merge")
builder.add_edge("process_right", "merge")

# 编译时自动并行化
graph = builder.compile()
# process_left 和 process_right 会并行执行
```

## 5. 状态机视角

LangGraph 的图本质是一个 **有状态的有限自动机**:

```
         ┌──────────┐
         │  START   │
         └────┬─────┘
              │
              ▼
         ┌──────────┐
     ┌───│  Parse   │───┐
     │   └────┬─────┘   │
     │        │         │
     │        ▼         │
     │   ┌──────────┐   │
     │   │ Validate │───┤ yes ──→ END
     │   └────┬─────┘
     │        │ no
     │        ▼
     │   ┌──────────┐
     └───│  Fix     │
         └──────────┘
```

这个自动机的关键属性：
- **每个状态** = State 的一个快照 (由 checkpointer 保存)
- **每次转移** = 一个节点执行完毕
- **转移函数** = 边的路由逻辑
- **最终状态** = 到达 END 节点或触发中断
