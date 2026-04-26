# LangGraph 流式输出与实时交互

## 1. 为什么需要流式输出

传统 LLM 调用的问题:
- 用户等待: 必须等整个执行完成才有响应
- 黑盒: 不知道 Agent 在做什么、是否卡住
- 不透明: 工具调用过程对前端不可见
- 体验差: 大任务可能要等几十秒

流式输出的价值:
- **渐进式交付**: 边计算边输出，降低首字延迟
- **过程可见**: 前端可以展示 Agent 的思考过程
- **中间态可用**: 可以展示工具调用结果、中间推理
- **用户体验**: 用户可以看到"Agent 正在思考/搜索/..."

## 2. 5 种 Stream Mode

### 2.1 模式对比总览

```
┌────────────────┬────────────────┬─────────────────────────┐
│ stream_mode    │ 返回内容       │ 适用场景                │
├────────────────┼────────────────┼─────────────────────────┤
│ "values"       │ 完整 State     │ 每次获取最新完整状态     │
│                │ (每步后)       │                         │
├────────────────┼────────────────┼─────────────────────────┤
│ "updates"      │ 增量更新       │ 展示每一步的变化         │
│                │ (每步后)       │                         │
├────────────────┼────────────────┼─────────────────────────┤
│ "messages"     │ LLM 消息片段   │ Token 级流式输出         │
│                │ (token 级)    │                         │
├────────────────┼────────────────┼─────────────────────────┤
│ "events"       │ 自定义事件     │ 细粒度控制               │
│                │ (事件驱动)    │                         │
├────────────────┼────────────────┼─────────────────────────┤
│ "custom"       │ writer 写入    │ 完全自定义               │
│                │ (手动)        │                         │
└────────────────┴────────────────┴─────────────────────────┘
```

### 2.2 "values" — 完整状态快照

```python
# 每次 yield 当前完整的 State
for chunk in graph.stream(input, stream_mode="values"):
    # chunk = AgentState (完整状态)
    print(chunk["messages"][-1])  # 打印最新消息
    display_state(chunk)          # 渲染当前状态
```

数据流:
```
Step 0: { messages: [user: "查询天气"] }
Step 1: { messages: [user: "查询天气", assistant: "让我查一下..."] }
Step 2: { messages: [user: ..., assistant: "让我查一下...", tool: "晴, 25°C"] }
Step 3: { messages: [user: ..., assistant: "让我查一下...", tool: ..., assistant: "今天晴天, 25°C"] }
```

适用:
- 前端展示完整的当前状态
- 状态变更时刷新 UI
- 调试和日志

### 2.3 "updates" — 增量更新

```python
# 每次 yield 当前节点产生的增量更新
for chunk in graph.stream(input, stream_mode="updates"):
    # chunk = { "node_name": { partial_state } }
    for node_name, node_update in chunk.items():
        print(f"Node [{node_name}] update: {node_update}")
```

数据流:
```
{"chatbot":  {"messages": [AIMessage("让我查一下...")]}}
{"tools":    {"messages": [ToolMessage("晴, 25°C")]}}
{"chatbot":  {"messages": [AIMessage("今天晴天, 25°C")]}}
```

适用:
- 展示每一步做了什么
- 前端按节点更新 UI 组件
- 构建 Activity Feed

### 2.4 "messages" — Token 级流式输出

```python
# 实时流式输出 LLM 生成的每个 token
for chunk in graph.stream(input, stream_mode="messages"):
    # chunk = (message, metadata)
    msg, metadata = chunk
    if hasattr(msg, "content") and msg.content:
        print(msg.content, end="", flush=True)  # 逐 token 输出
```

数据流:
```
今...天...晴...天...,....2...5...°...C...，...适...合...出...行...
```

适用:
- Chat 应用的打字机效果
- 降低用户感知延迟
- 与前端 WebSocket/SSE 配合

### 2.5 "events" — 自定义事件 (推荐)

```python
# 使用 astream_events 异步迭代
async for event in graph.astream_events(input, version="v2"):
    kind = event["event"]

    # on_chat_model_start — LLM 开始生成
    if kind == "on_chat_model_start":
        print(f"🤖 Agent 正在思考...")

    # on_chat_model_stream — LLM 输出 token
    elif kind == "on_chat_model_stream":
        token = event["data"]["chunk"].content
        if token:
            print(token, end="", flush=True)

    # on_chat_model_end — LLM 生成结束
    elif kind == "on_chat_model_end":
        print()  # 换行

    # on_tool_start — 工具调用开始
    elif kind == "on_tool_start":
        tool_name = event["name"]
        tool_input = event["data"].get("input")
        print(f"🔧 调用工具: {tool_name}({tool_input})")

    # on_tool_end — 工具调用结束
    elif kind == "on_tool_end":
        result = event["data"].get("output")
        print(f"✅ 工具返回: {result}")

    # on_chain_end — 节点执行结束
    elif kind == "on_chain_end":
        node_name = event["name"]
        print(f"📍 节点 [{node_name}] 完成")
```

适用:
- 最灵活的事件驱动模式
- 可以区分不同的事件类型
- 前端可以展示不同图标/状态
- 推荐用于生产级应用

### 2.6 "custom" — 完全自定义

```python
from langgraph.types import StreamWriter

def my_custom_node(state: AgentState, writer: StreamWriter) -> dict:
    # 手动写入自定义事件
    writer({"status": "thinking", "message": "正在分析问题..."})
    result = llm.invoke(state["messages"])
    writer({"status": "responding", "content": result.content})
    return {"messages": [result]}

# 编译时指定
graph = builder.compile()

# 执行时接收自定义写入
for chunk in graph.stream(input, stream_mode="custom"):
    print(f"Custom event: {chunk}")
```

## 3. 异步流式 API

```python
# 同步 API
for chunk in graph.stream(input, stream_mode="updates"):
    ...

# 异步 API (推荐用于 Web 服务)
async for chunk in graph.astream(input, stream_mode="updates"):
    ...

# 事件 API (仅异步)
async for event in graph.astream_events(input, version="v2"):
    ...
```

## 4. 前端集成示例

### 4.1 FastAPI + SSE

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import json

app = FastAPI()

@app.post("/chat")
async def chat(request: Request):
    body = await request.json()
    input_msg = body["message"]
    thread_id = body.get("thread_id", "default")

    config = {"configurable": {"thread_id": thread_id}}

    async def event_stream():
        async for event in graph.astream_events(
            {"messages": [("user", input_msg)]},
            version="v2",
            config=config
        ):
            kind = event["event"]
            data = {}

            if kind == "on_chat_model_stream":
                data = {
                    "type": "token",
                    "content": event["data"]["chunk"].content
                }
            elif kind == "on_tool_start":
                data = {
                    "type": "tool_start",
                    "name": event["name"],
                    "input": event["data"].get("input")
                }
            elif kind == "on_tool_end":
                data = {
                    "type": "tool_end",
                    "name": event["name"],
                    "output": event["data"].get("output")
                }

            if data:
                yield f"data: {json.dumps(data)}\n\n"

        yield "data: [DONE]\n\n"

    return StreamingResponse(
        event_stream(),
        media_type="text/event-stream"
    )
```

### 4.2 前端 (TypeScript)

```typescript
const response = await fetch("/chat", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ message: "今天天气怎样?" }),
});

const reader = response.body!.getReader();
const decoder = new TextDecoder();
let buffer = "";

while (true) {
  const { done, value } = await reader.read();
  if (done) break;

  buffer += decoder.decode(value);
  for (const line of buffer.split("\n\n")) {
    if (!line.startsWith("data: ")) continue;
    const data = line.slice(6);
    if (data === "[DONE]") continue;

    const event = JSON.parse(data);
    switch (event.type) {
      case "token":
        appendToken(event.content);  // 打字机效果
        break;
      case "tool_start":
        showToolCall(event.name);     // 显示工具调用
        break;
      case "tool_end":
        hideToolCall(event.name);     // 隐藏工具调用
        break;
    }
  }
  buffer = "";
}
```

## 5. 模式选择指南

```
需求                        推荐模式
─────────────────────────────────────
显示打字机效果              messages / events
展示 Agent 思考过程          events
每步刷新完整状态            values
仅关注增量变化              updates
自定义数据格式              custom
最简单实现                  values
生产级聊天应用              events + SSE
```
