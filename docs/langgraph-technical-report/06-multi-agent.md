# LangGraph 多 Agent 系统

## 1. 多 Agent 架构模式

LangGraph 提供了四种标准的多 Agent 模式，适用于不同的协作场景。

```
┌─────────────────────────────────────────────────────────┐
│                    多 Agent 模式                         │
├───────────────┬───────────────┬──────────┬──────────────┤
│ 流水线模式    │ 路由模式      │ 并行模式 │ 监督模式     │
│ (Pipeline)    │ (Routing)     │(Parallel)│(Supervisor)  │
└───────────────┴───────────────┴──────────┴──────────────┘
```

## 2. 流水线模式 (Pipeline)

### 2.1 概念

Agent 按固定顺序执行，每个 Agent 将结果传递给下一个:

```
User Input
    │
    ▼
┌───────────┐
│ Agent A   │  (研究)
│ (Researcher)
└─────┬─────┘
      │
      ▼
┌───────────┐
│ Agent B   │  (写作)
│ (Writer)  │
└─────┬─────┘
      │
      ▼
┌───────────┐
│ Agent C   │  (评审)
│ (Reviewer)│
└─────┬─────┘
      │
      ▼
   Final Output
```

### 2.2 实现

```python
from langgraph.graph import StateGraph, END, START
from typing import TypedDict, Annotated

class ResearchState(TypedDict):
    research: str       # 研究报告
    draft: str          # 初稿
    final: str          # 终稿
    topic: str          # 主题

# Node 1: 研究
def researcher(state: ResearchState) -> dict:
    report = research_llm.invoke(state["topic"])
    return {"research": report.content}

# Node 2: 写作
def writer(state: ResearchState) -> dict:
    draft = write_llm.invoke(
        f"Based on this research:\n{state['research']}\n"
        f"Write an article about {state['topic']}"
    )
    return {"draft": draft.content}

# Node 3: 评审
def reviewer(state: ResearchState) -> dict:
    final = review_llm.invoke(
        f"Review and improve:\n{state['draft']}"
    )
    return {"final": final.content}

# 构建图
builder = StateGraph(ResearchState)
builder.add_node("researcher", researcher)
builder.add_node("writer", writer)
builder.add_node("reviewer", reviewer)

builder.add_edge(START, "researcher")
builder.add_edge("researcher", "writer")
builder.add_edge("writer", "reviewer")
builder.add_edge("reviewer", END)

graph = builder.compile()

# 执行
result = graph.invoke({"topic": "LangGraph 多 Agent 架构"})
```

## 3. 路由模式 (Routing)

### 3.1 概念

由路由器 Agent 根据输入特征决定由哪个专业 Agent 处理:

```
          ┌──────────────┐
          │  Router      │
          │  (分类器)    │
          └──┬───┬───┬───┘
             │   │   │
    ┌────────┘   │   └────────┐
    ▼            ▼            ▼
┌────────┐ ┌────────┐ ┌────────┐
│ Agent 1│ │ Agent 2│ │ Agent 3│
│(编码)  │ │(写作)  │ │(数学)  │
└───┬────┘ └───┬────┘ └───┬────┘
    └──────────┼──────────┘
               ▼
           Final Output
```

### 3.2 实现

```python
class RoutedState(TypedDict):
    messages: Annotated[list, add_messages]
    result: str

def classify_intent(state: RoutedState) -> str:
    """判断用户意图"""
    response = classifier_llm.invoke(
        f"Classify: {state['messages'][-1].content}\n"
        "Options: coding, writing, math"
    )
    return response.content.strip()

def coding_agent(state: RoutedState) -> dict:
    result = coding_llm.invoke(state["messages"])
    return {"result": result.content}

def writing_agent(state: RoutedState) -> dict:
    result = writing_llm.invoke(state["messages"])
    return {"result": result.content}

def math_agent(state: RoutedState) -> dict:
    result = math_llm.invoke(state["messages"])
    return {"result": result.content}

builder = StateGraph(RoutedState)
builder.add_node("coding", coding_agent)
builder.add_node("writing", writing_agent)
builder.add_node("math", math_agent)

# 路由器根据意图分发
def route(state: RoutedState) -> str:
    intent = classify_intent(state)
    return intent  # "coding" | "writing" | "math"

builder.add_conditional_edges(START, route)
builder.add_edge("coding", END)
builder.add_edge("writing", END)
builder.add_edge("math", END)

graph = builder.compile()
```

## 4. 并行模式 (Parallel)

### 4.1 概念

多个 Agent 同时执行，结果合并:

```
         ┌───────────────────┐
         │      Split        │
         └──┬──────────┬─────┘
            │          │
   ┌────────┘          └────────┐
   ▼                            ▼
┌──────────┐               ┌──────────┐
│ Agent A  │               │ Agent B  │
│ (搜索)   │               │ (代码)   │
└────┬─────┘               └────┬─────┘
     │                          │
     └───────────┬──────────────┘
                 ▼
          ┌─────────────┐
          │    Merge     │
          │  (整合)      │
          └──────┬───────┘
                 ▼
             Final Output
```

### 4.2 实现

```python
class ParallelState(TypedDict):
    query: str
    search_result: str
    code_result: str
    merged: str

def search_agent(state: ParallelState) -> dict:
    result = search_llm.invoke(state["query"])
    return {"search_result": result.content}

def code_agent(state: ParallelState) -> dict:
    result = code_llm.invoke(state["query"])
    return {"code_result": result.content}

def merge_agent(state: ParallelState) -> dict:
    merged = merge_llm.invoke(
        f"Search results:\n{state['search_result']}\n"
        f"Code analysis:\n{state['code_result']}\n"
        f"Combine into a comprehensive answer."
    )
    return {"merged": merged.content}

builder = StateGraph(ParallelState)
builder.add_node("search", search_agent)
builder.add_node("code", code_agent)
builder.add_node("merge", merge_agent)

# START 同时到两个节点 (并行)
builder.add_edge(START, "search")
builder.add_edge(START, "code")
# 两个完成后都到 merge
builder.add_edge("search", "merge")
builder.add_edge("code", "merge")
builder.add_edge("merge", END)

graph = builder.compile()
```

## 5. 监督模式 (Supervisor)

### 5.1 概念

由 Supervisor Agent 调度多个 Worker Agent，动态决定下一步执行哪个:

```
              ┌────────────────────┐
              │   Supervisor       │
              │   (管理者)         │
              │  "下一步做什么?"    │
              └──┬───┬───┬───┬───┐
                 │   │   │   │
        ┌────────┘   │   │   └────────┐
        ▼            ▼   ▼            ▼
   ┌────────┐ ┌────────┐ ┌────────┐
   │Worker 1│ │Worker 2│ │Worker 3│
   │(研究)  │ │(编码)  │ │(评审)  │
   └───┬────┘ └───┬────┘ └───┬────┘
       └──────────┼──────────┘
                  │
                  ▼
          回到 Supervisor
```

### 5.2 实现

```python
from typing import Literal
from langgraph.types import Command

class SupervisorState(TypedDict):
    messages: Annotated[list, add_messages]
    next: str

def supervisor(state: SupervisorState) -> Command:
    """Supervisor 决定下一步"""
    response = supervisor_llm.invoke(
        [SystemMessage(content="""你负责管理一个团队。
          可用 worker: research, coding, review.
          根据当前进度决定下一步。
          回答: research, coding, review, 或 FINISH"""),
         *state["messages"]]
    )

    decision = response.content.strip()
    if decision == "FINISH":
        return Command(goto=END)
    return Command(goto=decision)

def research_worker(state: SupervisorState) -> dict:
    result = research_llm.invoke(state["messages"])
    return {"messages": [result], "next": "supervisor"}

def coding_worker(state: SupervisorState) -> dict:
    result = coding_llm.invoke(state["messages"])
    return {"messages": [result], "next": "supervisor"}

def review_worker(state: SupervisorState) -> dict:
    result = review_llm.invoke(state["messages"])
    return {"messages": [result], "next": "supervisor"}

builder = StateGraph(SupervisorState)
builder.add_node("supervisor", supervisor)
builder.add_node("research", research_worker)
builder.add_node("coding", coding_worker)
builder.add_node("review", review_worker)

# Supervisor 决定下一步
builder.add_conditional_edges("supervisor",
    lambda state: state["next"],  # 由 supervisor 的 Command.goto 决定
    {"research": "research", "coding": "coding", "review": "review"}
)

# Worker 完成后回到 Supervisor
builder.add_edge("research", "supervisor")
builder.add_edge("coding", "supervisor")
builder.add_edge("review", "supervisor")

builder.add_edge(START, "supervisor")

graph = builder.compile()
```

## 6. 子图嵌套 (Subgraph)

### 6.1 概念

一个图可以作为另一个图的节点，形成层级结构:

```
┌─────────────────────────────────────┐
│           Parent Graph              │
│                                     │
│  START ──> [Plan] ──┐               │
│                     ▼               │
│              ┌───────────────┐      │
│              │  Child Graph  │      │
│              │ (子图)        │      │
│              │               │      │
│              │  START → A → B│      │
│              │         ↓     │      │
│              │         C → END      │
│              └───────┬───────┘      │
│                      ▼              │
│              [Deliver] ──> END      │
│                                     │
└─────────────────────────────────────┘
```

### 6.2 实现

```python
# 1. 定义子图
def step_a(state: ChildState) -> dict:
    return {"step_a_result": "done"}

def step_b(state: ChildState) -> dict:
    return {"step_b_result": "done"}

child_builder = StateGraph(ChildState)
child_builder.add_node("a", step_a)
child_builder.add_node("b", step_b)
child_builder.add_edge(START, "a")
child_builder.add_edge("a", "b")
child_builder.add_edge("b", END)

child_graph = child_builder.compile()

# 2. 将子图作为父图的节点
def execute_pipeline(state: ParentState) -> dict:
    # 调用子图
    result = child_graph.invoke({
        "input": state["task"]
    })
    return {"pipeline_result": result}

parent_builder = StateGraph(ParentState)
parent_builder.add_node("plan", plan_task)
parent_builder.add_node("pipeline", execute_pipeline)  # 子图!
parent_builder.add_node("deliver", deliver_result)

parent_builder.add_edge(START, "plan")
parent_builder.add_edge("plan", "pipeline")
parent_builder.add_edge("pipeline", "deliver")
parent_builder.add_edge("deliver", END)

parent_graph = parent_builder.compile()
```

## 7. 多 Agent 通信模式

### 7.1 共享状态 (Shared State)

```python
# 所有 Agent 共享同一个 State
class TeamState(TypedDict):
    messages: Annotated[list, add_messages]
    research_notes: str
    code_output: str
    review_comments: str
```

优点: 简单，适合小规模
缺点: 状态膨胀，耦合度高

### 7.2 消息传递 (Message Passing)

```python
# 通过消息队列通信
class MessageState(TypedDict):
    messages: Annotated[list, add_messages]
    # 每个 Agent 通过消息交流，不直接读写其他 Agent 的 State

def agent_a(state: MessageState) -> dict:
    response = llm.invoke(state["messages"])
    return {"messages": [response]}  # 通过消息传递结果
```

优点: 解耦，灵活
缺点: 需要更多调度

### 7.3 混合模式 (Hybrid)

```python
# 共享状态 + 专用字段
class HybridState(TypedDict):
    messages: Annotated[list, add_messages]  # 所有 Agent 共享
    research: str                             # 仅供 Researcher
    code: str                                 # 仅供 Coder
    review: str                               # 仅供 Reviewer
```

## 8. 选型指南

```
场景                      推荐模式
─────────────────────────────────────
固定步骤流水线            流水线 (Pipeline)
不同类型请求路由          路由 (Routing)
独立任务并行              并行 (Parallel)
复杂任务, 动态决策        监督 (Supervisor)
嵌套复杂流程              子图 (Subgraph)
需要多 Agent 协作讨论      Supervisor + Subgraph
```
