# LangGraph 人类在环 (Human-in-the-Loop)

## 1. 概念概述

人类在环是指在 Agent 的执行流程中嵌入人工干预点，允许人类在关键节点:
- **查看**: Agent 的当前状态和决策
- **批准/拒绝**: 对 Agent 的下一步行动进行审批
- **编辑**: 修改 Agent 的状态或参数
- **补充**: 提供额外信息引导 Agent

```
Agent 自主执行                    Agent ─┐ 等待人类审批 ── 人类批准 ── Agent 继续执行
                                      │
                                      ├── 人类拒绝 ── Agent 重新规划
                                      │
                                      └── 人类编辑 ── Agent 使用新参数继续
```

## 2. 中断点 (Interrupt)

### 2.1 interrupt() 函数

```python
from langgraph.types import interrupt

class AgentState(TypedDict):
    messages: Annotated[list, add_messages]
    result: str
    action: str

def approve_action(state: AgentState) -> dict:
    # 暂停执行，等待人类输入
    user_input = interrupt(
        {"question": "请确认是否发送以下邮件?",
         "draft": state["messages"][-1].content}
    )
    # user_input 是人类通过 update_state 传入的值
    if user_input == "approved":
        return {"status": "sent"}
    else:
        return {"status": "rejected"}

builder.add_node("approve", approve_action)
builder.add_edge(START, "approve")
builder.add_edge("approve", END)

graph = builder.compile(checkpointer=MemorySaver())
```

### 2.2 中断生命周期

```
1. Agent 执行到 interrupt() 处
   └─→ 图暂停，保存检查点
   └─→ 返回 interrupt 的参数给调用方

2. 外部系统 (或前端) 获取中断信息
   └─→ 展示给人类用户
   └─→ 等待人类操作

3. 人类操作后，通过 update_state 传入决策
   └─→ interrupt() 返回人类输入
   └─→ Agent 继续执行

4. Agent 执行到 END 或下一个 interrupt
```

### 2.3 完整交互流程

```python
# 第一次: Agent 执行到 interrupt 处暂停
config = {"configurable": {"thread_id": "approval-1"}}
for chunk in graph.stream(input_draft, config=config):
    print(chunk)
# 输出: {"approve": {"question": "请确认是否发送?", ...}}

# 检查当前状态
state = graph.get_state(config)
print(state.next)  # ("approve",) — 当前在 approve 节点暂停

# 人类批准: 通过 update_state 传入决策
graph.update_state(
    config,
    {"user_response": "approved"},  # 作为 interrupt 的返回值
    as_node="approve"  # 指定从哪个节点更新
)

# 继续执行
for chunk in graph.stream(None, config=config):
    print(chunk)
# 输出: {...} — 继续执行剩余部分
```

## 3. 预检查点 (Pre-checkpoint)

### 3.1 概念

默认情况下，检查点在节点执行后保存。通过 `interrupt_before` 参数，可以在节点执行前暂停:

```
正常模式:                          预检查点模式:
Node A ──> Node B (执行)            Node A ──> Node B (暂停! 等人类审批)
                    │                              │
                    ▼                              ▼
              Checkpoint saved              人类审批后才执行
                                            └─> Node B 执行
                                            └─> Checkpoint saved
```

### 3.2 interrupt_before

```python
# 在执行 approve 节点之前暂停
graph = builder.compile(
    checkpointer=MemorySaver(),
    interrupt_before=["approve"]  # 在 approve 节点执行前暂停
)

# 第一次 invoke: 执行到 START 后暂停 (不执行 approve)
config = {"configurable": {"thread_id": "1"}}
result = graph.invoke(input_draft, config=config)
# result 不包含 approve 节点的输出

# 检查: 当前在 approve 节点等待执行
state = graph.get_state(config)
print(state.next)  # ("approve",)

# 人类确认后再执行
graph.invoke(None, config=config)  # 继续执行
```

### 3.3 interrupt_after

```python
# 在执行完 approve 节点后暂停 (无论是否还有后续节点)
graph = builder.compile(
    checkpointer=MemorySaver(),
    interrupt_after=["approve"]  # approve 执行完后暂停
)
```

### 3.4 interrupt_before vs interrupt_after

```
interrupt_before(["node_x"]):    interrupt_after(["node_x"]):
- 在进入 node_x 之前暂停       - 在离开 node_x 之后暂停
- node_x 还没开始执行          - node_x 已经执行完毕
- 适合: 审批 node_x 是否执行    - 适合: 检查 node_x 的输出
- 人类决定是否执行 node_x      - 人类检查 node_x 的结果
```

## 4. 编辑历史状态

### 4.1 编辑并重新执行

```python
# 1. 查看历史
history = list(checkpointer.list(config))

# 2. 找到要编辑的检查点
target = history[1]  # 回到第 2 步

# 3. 从目标检查点获取当前状态
state = graph.get_state(target.config)

# 4. 编辑 State (替换部分值)
edited_state = state.values.copy()
edited_state["messages"][-1] = ("assistant", "重新写一个更好的回答")

# 5. 更新状态
graph.update_state(
    config,
    edited_state,
    as_node=state.next[0]  # 从当前暂停的节点更新
)

# 6. 从编辑后的状态继续执行
graph.invoke(None, config=config)
```

### 4.2 状态编辑的典型场景

```python
# 场景 1: 修改 LLM 的参数
graph.update_state(
    config,
    {"temperature": 0.1, "model": "gpt-4"},
    as_node="chatbot"
)

# 场景 2: 修正工具执行结果
graph.update_state(
    config,
    {"tool_result": "修正后的结果"},
    as_node="tool_executor"
)

# 场景 3: 添加上下文信息
graph.update_state(
    config,
    {"context": {"user_role": "admin", "dept": "finance"}},
    as_node="chatbot"
)
```

## 5. 审核批准流程

### 5.1 完整的审批示例

```python
from langgraph.types import interrupt
from langgraph.checkpoint.memory import MemorySaver

class CodeReviewState(TypedDict):
    messages: Annotated[list, add_messages]
    code: str
    review_result: str
    approved: bool

def generate_code(state: CodeReviewState) -> dict:
    """AI 生成代码"""
    code = llm.invoke(f"Write code for: {state['messages'][-1].content}")
    return {"code": code.content}

def review_code(state: CodeReviewState) -> dict:
    """人类审核代码"""
    decision = interrupt({
        "type": "code_review",
        "code": state["code"],
        "question": "请审核以下代码，批准或提出修改意见:"
    })

    if decision == "approved":
        return {"approved": True, "review_result": "Approved"}
    elif decision == "revise":
        return {"approved": False, "review_result": "Needs revision"}
    else:
        # 人类提供了修改意见，AI 重新生成
        return {"approved": False, "review_result": f"Revisions: {decision}"}

def incorporate_feedback(state: CodeReviewState) -> dict:
    """根据反馈修改代码"""
    new_code = llm.invoke(
        f"Original code:\n{state['code']}\n\n"
        f"Feedback:\n{state['review_result']}\n\n"
        f"Please revise the code."
    )
    return {"code": new_code.content}

# 构建图
builder = StateGraph(CodeReviewState)
builder.add_node("generate", generate_code)
builder.add_node("review", review_code)
builder.add_node("revise", incorporate_feedback)

builder.add_edge(START, "generate")
builder.add_edge("generate", "review")

def route_review(state: CodeReviewState) -> str:
    if state["approved"]:
        return "end"
    else:
        return "revise"

builder.add_conditional_edges("review", route_review, {
    "end": END,
    "revise": "revise"
})
builder.add_edge("revise", "review")  # 循环回到审核

# 编译: 在 review 节点前暂停
graph = builder.compile(
    checkpointer=MemorySaver(),
    interrupt_before=["review"]
)

# 执行
config = {"configurable": {"thread_id": "review-1"}}
for chunk in graph.stream(
    {"messages": [("user", "写一个排序函数")]},
    config=config
):
    print(chunk)

# 人类审核: 批准
graph.invoke(None, config=config)  # 继续，但 review 节点仍会等待

# 通过 update_state 传入审批结果
graph.update_state(config, {"user_decision": "approved"}, as_node="review")

# 继续执行
graph.invoke(None, config=config)
```

## 6. 前端集成

### 6.1 审批 UI 工作流

```
前端                     后端                       Agent
  │                       │                         │
  │── POST /chat ──────────>│                         │
  │                       │── invoke ────────────────>│
  │                       │                         │
  │                       │<── interrupt ─────────────│
  │<── {type: "review",   │   (code, question)        │
  │    code: "..."}───────│                         │
  │                       │                         │
  │  [用户查看代码]         │                         │
  │  [用户点击 批准/修改]   │                         │
  │── POST /approve ───────>│                         │
  │                       │── update_state ──────────>│
  │                       │                         │
  │<── {result: "..."}────│<── continue ──────────────│
```

### 6.2 FastAPI 完整示例

```python
@app.post("/chat")
async def chat(request: Request):
    body = await request.json()
    config = {"configurable": {"thread_id": body["thread_id"]}}

    # 发送用户消息
    input_data = {"messages": [("user", body["message"])]}

    events = []
    async for event in graph.astream_events(input_data, version="v2", config=config):
        # 收集事件...
        pass

    # 检查是否有中断
    state = graph.get_state(config)
    if state.next:  # 有节点在等待
        return {"status": "awaiting_approval",
                "node": state.next[0],
                "values": state.values}

    return {"status": "complete", "result": events}


@app.post("/approve")
async def approve(request: Request):
    body = await request.json()
    config = {"configurable": {"thread_id": body["thread_id"]}}

    # 传入人类决策
    graph.update_state(
        config,
        {"user_decision": body["decision"]},
        as_node=body["node"]
    )

    # 继续执行
    result = graph.invoke(None, config=config)
    return {"status": "complete", "result": result}
```

## 7. 最佳实践

```python
# ✅ 推荐
# 1. 用有意义的 interrupt 参数描述待决策内容
interrupt({"type": "email_send", "content": draft, "recipients": ["a@b.com"]})

# 2. 用 interrupt_before 做预审批
builder.compile(checkpointer=..., interrupt_before=["dangerous_action"])

# 3. 检查 state.next 判断是否有中断
state = graph.get_state(config)
if state.next:
    # 需要人类介入
    ...

# ❌ 避免
# 1. 不要在 interrupt 后假设 State 已被修改
# interrupt 只暂停，不自动修改 State

# 2. 不要在多节点场景使用相同 interrupt 参数
# 不同的 interrupt 节点应有不同的参数结构

# 3. 不要在不使用 checkpointer 的情况下使用 interrupt
# interrupt 依赖检查点来暂停和恢复
```
