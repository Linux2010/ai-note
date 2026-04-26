# LangGraph 完整技术方案与示例代码

## 1. 端到端示例: 研究分析助手

一个完整的 Agent 应用: 接收用户研究主题 → 搜索 → 分析 → 生成报告 → 人类审核 → 发布

### 1.1 定义 State

```python
from typing import TypedDict, Annotated, Literal
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, END, START
from langgraph.graph.message import add_messages
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.types import interrupt, StreamWriter
import operator

class ResearchState(TypedDict):
    # LLM 对话历史 (合并模式)
    messages: Annotated[list, add_messages]

    # 研究主题
    topic: str

    # 搜索结果
    search_results: Annotated[list, operator.add]

    # 分析摘要
    analysis: str

    # 报告草稿
    draft: str

    # 审核结果
    approved: bool | None

    # 发布内容
    final_output: str

    # 错误信息
    errors: Annotated[list, operator.add]
```

### 1.2 定义节点

```python
# ───────── 搜索节点 ─────────
def search_topic(state: ResearchState, writer: StreamWriter) -> dict:
    """搜索研究主题"""
    writer({"status": "searching", "message": f"正在搜索: {state['topic']}"})

    # 调用搜索工具 (这里用模拟实现)
    results = search_tool.invoke(state["topic"])
    writer({"status": "search_done", "count": len(results)})

    return {"search_results": [str(results)]}


# ───────── 分析节点 ─────────
def analyze_results(state: ResearchState, writer: StreamWriter) -> dict:
    """分析搜索结果"""
    writer({"status": "analyzing", "message": "正在分析收集的信息..."})

    analysis_prompt = f"""
    基于以下搜索结果，分析研究主题: {state['topic']}

    搜索结果:
    {chr(10).join(state['search_results'])}

    请提供:
    1. 关键发现
    2. 争议点
    3. 数据支撑
    """
    analysis = llm.invoke(analysis_prompt)
    writer({"status": "analysis_done"})

    return {"analysis": analysis.content}


# ───────── 写作节点 ─────────
def write_draft(state: ResearchState, writer: StreamWriter) -> dict:
    """撰写报告草稿"""
    writer({"status": "writing", "message": "正在撰写研究报告..."})

    draft_prompt = f"""
    研究主题: {state['topic']}

    分析结果:
    {state['analysis']}

    请撰写一份结构化的研究报告，包含:
    - 摘要
    - 引言
    - 分析方法
    - 发现与讨论
    - 结论
    """
    draft = llm.invoke(draft_prompt)
    writer({"status": "draft_done"})

    return {"draft": draft.content}


# ───────── 审核节点 ─────────
def human_review(state: ResearchState) -> dict:
    """人类审核报告"""
    decision = interrupt({
        "type": "draft_review",
        "topic": state["topic"],
        "draft": state["draft"],
        "question": "请审核这份研究报告:"
    })

    if decision == "approve":
        return {"approved": True}
    elif decision == "revise":
        return {"approved": False}
    else:
        # 人类提供了修改意见
        return {"approved": False, "revision_notes": decision}


# ───────── 修改节点 ─────────
def revise_draft(state: ResearchState, writer: StreamWriter) -> dict:
    """根据反馈修改草稿"""
    writer({"status": "revising", "message": "正在根据反馈修改..."})

    revision_prompt = f"""
    原草稿:
    {state['draft']}

    审核意见:
    {state.get('revision_notes', '请改进报告质量')}

    请根据意见修改草稿。
    """
    revised = llm.invoke(revision_prompt)
    return {"draft": revised.content, "revision_notes": None}


# ───────── 发布节点 ─────────
def publish(state: ResearchState, writer: StreamWriter) -> dict:
    """发布最终报告"""
    writer({"status": "publishing", "message": "正在发布报告..."})

    final = f"""
    # 研究报告: {state['topic']}

    {state['draft']}

    ---
    生成时间: {datetime.now().isoformat()}
    """
    writer({"status": "published"})
    return {"final_output": final}
```

### 1.3 构建图

```python
builder = StateGraph(ResearchState)

# 注册所有节点
builder.add_node("search", search_topic)
builder.add_node("analyze", analyze_results)
builder.add_node("write", write_draft)
builder.add_node("review", human_review)
builder.add_node("revise", revise_draft)
builder.add_node("publish", publish)

# 连接: 搜索 → 分析 → 写作 → 审核
builder.add_edge(START, "search")
builder.add_edge("search", "analyze")
builder.add_edge("analyze", "write")
builder.add_edge("write", "review")

# 审核路由
def route_review(state: ResearchState) -> Literal["publish", "revise"]:
    return "publish" if state["approved"] else "revise"

builder.add_conditional_edges("review", route_review, {
    "publish": "publish",
    "revise": "revise"
})

# 修改后回到审核
builder.add_edge("revise", "review")
builder.add_edge("publish", END)
```

### 1.4 编译与执行

```python
from langgraph.checkpoint.postgres import PostgresSaver

# 使用 PostgreSQL 持久化
checkpointer = PostgresSaver.from_conn_string(DB_URL)
checkpointer.setup()

# 编译: 在审核节点前暂停
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["review"]
)

# ─── 第一次: 生成草稿 ───
config = {"configurable": {"thread_id": "research-001"}}
input_data = {"topic": "LangGraph 在 Agent 开发中的应用"}

# 流式执行
async def run_research():
    async for event in graph.astream_events(
        input_data, version="v2", config=config
    ):
        kind = event["event"]
        if kind == "on_chain_end":
            node = event["name"]
            print(f"✅ 节点完成: {node}")
        elif kind == "on_custom_event":
            data = event["data"]
            print(f"📡 {data['status']}: {data.get('message', '')}")

    # 检查是否中断
    state = graph.get_state(config)
    if state.next:
        print(f"\n⏸️ 等待审核: 节点 {state.next[0]}")

# ─── 审核: 人类批准 ───
async def approve_research():
    graph.update_state(
        config,
        {"approved": True},
        as_node="review"
    )
    for chunk in graph.stream(None, config=config):
        print(chunk)

# ─── 审核: 人类要求修改 ───
async def request_revision():
    graph.update_state(
        config,
        {"approved": False, "revision_notes": "请增加更多数据支撑"},
        as_node="review"
    )
    for chunk in graph.stream(None, config=config):
        print(chunk)
```

## 2. 端到端示例: 代码审查 Agent

```python
class CodeReviewState(TypedDict):
    messages: Annotated[list, add_messages]
    code: str
    lint_results: str
    security_issues: list
    suggestions: list
    review_report: str

def parse_code(state: CodeReviewState) -> dict:
    """提取代码"""
    code = extract_code_from_messages(state["messages"])
    return {"code": code}

def lint_code(state: CodeReviewState) -> dict:
    """运行 Lint"""
    results = run_linter(state["code"])
    return {"lint_results": results}

def security_check(state: CodeReviewState) -> dict:
    """安全扫描"""
    issues = scan_security(state["code"])
    return {"security_issues": issues}

def generate_report(state: CodeReviewState) -> dict:
    """生成审查报告"""
    report = report_llm.invoke(
        f"Code:\n{state['code']}\n"
        f"Lint:\n{state['lint_results']}\n"
        f"Security:\n{state['security_issues']}"
    )
    return {"review_report": report.content}

builder = StateGraph(CodeReviewState)
builder.add_node("parse", parse_code)
builder.add_node("lint", lint_code)
builder.add_node("security", security_check)
builder.add_node("report", generate_report)

builder.add_edge(START, "parse")
builder.add_edge("parse", "lint")
builder.add_edge("parse", "security")  # 并行!
builder.add_edge("lint", "report")
builder.add_edge("security", "report")
builder.add_edge("report", END)

graph = builder.compile()
```

## 3. 端到端示例: 客服 Agent (带持久化)

```python
class SupportState(TypedDict):
    messages: Annotated[list, add_messages]
    ticket_id: str
    ticket_status: str
    customer_info: dict
    escalation: bool

def classify_ticket(state: SupportState) -> dict:
    """分类工单"""
    classification = classify_llm.invoke(
        f"Classify: {state['messages'][-1].content}\n"
        "Categories: billing, technical, complaint, general"
    )
    return {"ticket_category": classification.content}

def lookup_customer(state: SupportState, store: BaseStore) -> dict:
    """查询客户信息"""
    user_id = get_user_id_from_config()
    info = store.get(["customers", user_id], "profile")
    return {"customer_info": info.value if info else {}}

def answer_question(state: SupportState) -> dict:
    """回答问题"""
    system_prompt = build_system_prompt(
        category=state["ticket_category"],
        customer=state["customer_info"]
    )
    response = support_llm.invoke(
        [SystemMessage(content=system_prompt)] + state["messages"]
    )
    return {"messages": [response]}

def escalate(state: SupportState) -> dict:
    """升级工单"""
    return {"ticket_status": "escalated", "escalation": True}

def resolve_ticket(state: SupportState) -> dict:
    """解决工单"""
    return {"ticket_status": "resolved"}

builder = StateGraph(SupportState)
builder.add_node("classify", classify_ticket)
builder.add_node("lookup", lookup_customer)
builder.add_node("answer", answer_question)
builder.add_node("escalate", escalate)
builder.add_node("resolve", resolve_ticket)

builder.add_edge(START, "classify")
builder.add_edge("classify", "lookup")
builder.add_edge("lookup", "answer")

def route_answer(state: SupportState) -> str:
    confidence = check_confidence(state["messages"][-1])
    if confidence < 0.5:
        return "escalate"
    return "resolve"

builder.add_conditional_edges("answer", route_answer, {
    "escalate": "escalate",
    "resolve": "resolve"
})
builder.add_edge("escalate", END)
builder.add_edge("resolve", END)

# 编译: 持久化 + 共享存储
checkpointer = PostgresSaver.from_conn_string(DB_URL)
store = PostgresStore.from_conn_string(DB_URL)

graph = builder.compile(
    checkpointer=checkpointer,
    store=store
)
```

## 4. 生产架构

### 4.1 典型部署架构

```
┌─────────────────────────────────────────────────────────┐
│                      Client (Web/Mobile)                │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│              API Gateway / Load Balancer                │
└──────────────────────┬──────────────────────────────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
     ┌────────┐  ┌────────┐  ┌────────┐
     │ FastAPI│  │ FastAPI│  │ FastAPI│
     │ Node 1 │  │ Node 2 │  │ Node 3 │
     │ (Gunicorn│ │ (Gunicorn│ │ (Gunicorn│
     │  + uvicorn)│ │  + uvicorn)│ │  + uvicorn)│
     └───┬────┘  └───┬────┘  └───┬────┘
         └───────────┼───────────┘
                     │
         ┌───────────┼───────────┐
         ▼           ▼           ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │Postgres  │ │  Redis   │ │ LangSmith│
   │(Checkpts)│ │ (Cache)  │ │(Observability)
   └──────────┘ └──────────┘ └──────────┘
```

### 4.2 FastAPI 服务实现

```python
from fastapi import FastAPI, BackgroundTasks
from fastapi.responses import StreamingResponse
import json

app = FastAPI()
checkpointer = PostgresSaver.from_conn_string(DB_URL)
graph = builder.compile(checkpointer=checkpointer)

@app.post("/chat/{thread_id}")
async def chat(thread_id: str, request: Request):
    body = await request.json()
    config = {"configurable": {"thread_id": thread_id}}
    input_data = {"messages": [("user", body["message"])]}

    async def event_stream():
        async for event in graph.astream_events(
            input_data, version="v2", config=config
        ):
            kind = event["event"]
            if kind == "on_chat_model_stream":
                content = event["data"]["chunk"].content
                if content:
                    yield f"data: {json.dumps({'type': 'token', 'content': content})}\n\n"
            elif kind == "on_tool_start":
                yield f"data: {json.dumps({'type': 'tool_start', 'name': event['name']})}\n\n"
            elif kind == "on_tool_end":
                yield f"data: {json.dumps({'type': 'tool_end', 'name': event['name']})}\n\n"

        # 检查中断
        state = graph.get_state(config)
        if state.next:
            yield f"data: {json.dumps({'type': 'interrupt', 'node': state.next[0], 'values': state.values})}\n\n"

        yield "data: [DONE]\n\n"

    return StreamingResponse(event_stream(), media_type="text/event-stream")


@app.post("/resolve/{thread_id}")
async def resolve_interrupt(thread_id: str, request: Request):
    body = await request.json()
    config = {"configurable": {"thread_id": thread_id}}

    graph.update_state(
        config,
        body.get("update", {}),
        as_node=body["node"]
    )

    result = graph.invoke(None, config=config)
    return {"status": "ok", "result": result}
```

### 4.3 Docker 部署

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["gunicorn", "app:app",
     "--worker-class", "uvicorn.workers.UvicornWorker",
     "--workers", "4",
     "--bind", "0.0.0.0:8000"]
```

```yaml
# docker-compose.yml
version: "3.8"
services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DB_URL=postgresql://langgraph:password@postgres:5432/langgraph
      - REDIS_URL=redis://redis:6379
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: langgraph
      POSTGRES_PASSWORD: password
      POSTGRES_DB: langgraph
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine

volumes:
  pgdata:
```

## 5. 性能优化

### 5.1 并行执行优化

```python
# 让独立节点自动并行 (编译器优化)
# LangGraph 在 compile 时自动检测并行机会:
# 如果两个节点都依赖同一个上游节点，它们会并行执行
builder.add_edge("split", "task_a")
builder.add_edge("split", "task_b")
builder.add_edge("task_a", "merge")
builder.add_edge("task_b", "merge")

# task_a 和 task_b 会并行执行，无需额外配置
graph = builder.compile()
```

### 5.2 批量处理

```python
# 使用 batch 批量执行
configs = [
    {"configurable": {"thread_id": f"thread-{i}"}}
    for i in range(100)
]
inputs = [{"topic": f"topic-{i}"} for i in range(100)]

results = graph.batch(zip(inputs, configs))
```

### 5.3 连接池优化

```python
from psycopg_pool import ConnectionPool

# 生产推荐: 连接池
pool = ConnectionPool(
    conninfo=DB_URL,
    min_size=5,
    max_size=20,
    open=False,  # 延迟打开
)
pool.open()

checkpointer = PostgresSaver(pool)
graph = builder.compile(checkpointer=checkpointer)
```

## 6. 错误处理

### 6.1 节点级错误处理

```python
def robust_node(state: AgentState) -> dict:
    try:
        result = call_external_service(state)
        return {"result": result}
    except TimeoutError:
        return {"error": "timeout", "status": "retry"}
    except ValueError as e:
        return {"error": str(e), "status": "invalid_input"}
    except Exception as e:
        # 兜底: 返回错误状态而不是抛出异常
        return {"error": f"unexpected: {e}", "status": "failed"}
```

### 6.2 重试策略

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=30)
)
def call_llm_with_retry(messages):
    return llm.invoke(messages)

def resilient_node(state: AgentState) -> dict:
    response = call_llm_with_retry(state["messages"])
    return {"messages": [response]}
```

## 7. 安全最佳实践

```python
# 1. 不要将敏感数据放入 State
# ❌ 避免
class BadState(TypedDict):
    api_key: str  # 会被检查点持久化!
    password: str # 同样会被持久化!

# ✅ 使用 config 传递
def secure_node(state: AgentState, config: RunnableConfig) -> dict:
    api_key = config["configurable"]["api_key"]
    ...

# 2. 限制 State 大小 (防止注入攻击)
MAX_MESSAGES = 50
def trim_messages(state: AgentState) -> dict:
    if len(state["messages"]) > MAX_MESSAGES:
        return {"messages": state["messages"][-MAX_MESSAGES:]}
    return {}

# 3. 验证用户输入
from pydantic import BaseModel, field_validator

class UserInput(BaseModel):
    message: str

    @field_validator("message")
    @classmethod
    def validate(cls, v: str) -> str:
        if len(v) > 4096:
            raise ValueError("Message too long")
        return v.strip()
```

## 8. 可观测性

### 8.1 LangSmith 集成

```python
import os

# 启用 LangSmith
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your-api-key"
os.environ["LANGCHAIN_PROJECT"] = "my-langgraph-project"

# 自动追踪所有图执行
graph = builder.compile(checkpointer=checkpointer)

# 执行时自动记录
# - 每次节点执行
# - 每次 LLM 调用
# - 每次工具调用
# - 完整的 State 变更历史
result = graph.invoke(input_data, config=config)
```

### 8.2 自定义指标

```python
import time
from collections import defaultdict

metrics = defaultdict(list)

class MetricsNode:
    """包装节点，记录执行时间"""
    def __init__(self, node_func, node_name):
        self.node_func = node_func
        self.node_name = node_name

    def __call__(self, state):
        start = time.time()
        result = self.node_func(state)
        duration = time.time() - start
        metrics[self.node_name].append(duration)
        print(f"[{self.node_name}] took {duration:.2f}s")
        return result

# 使用
builder.add_node("search", MetricsNode(search_topic, "search"))
```
