---
id: summary-langgraph-interaction
type: summary
title: "LangGraph 流式交互与人类在环"
status: draft
updated: 2026-08-14
owner: "llm-wiki-expert"
sources:
  - ../../raw/sources/docs/langgraph-technical-report/04-streaming.md
  - ../../raw/sources/docs/langgraph-technical-report/05-human-in-the-loop.md
---

# LangGraph 流式交互与人类在环

## 概览

流式输出让 Agent 的执行过程对用户可见，人类在环则允许用户在关键节点暂停、检查、修改或批准执行。二者结合后，前端不仅展示生成文本，还要承载活动事件、工具状态、审批请求和恢复操作。

## 流式机制

| 模式 | 输出内容 | 适用场景 |
|---|---|---|
| `values` | 每一步的完整 State | 调试、完整状态面板 |
| `updates` | 节点产生的增量更新 | 活动流、步骤进度 |
| `messages` | Token 或消息片段 | 聊天打字机体验 |
| `events` | 带类型的运行事件 | 生产级事件协议 |
| `custom` | 节点主动写出的自定义数据 | 业务进度与专用 UI |

流式协议的核心价值是降低首个反馈的等待时间，并把模型、工具和路由过程转化为可观测事件。客户端应根据事件类型渲染，而不是把所有数据都当作文本拼接。

## 人类在环机制

`interrupt()` 可以在节点内部产生中断并等待外部输入；编译时的 `interrupt_before` 和 `interrupt_after` 则在指定节点前后建立稳定检查点。恢复时必须使用同一个 Thread 配置，让执行引擎找到正确的历史状态。

典型生命周期包括：执行到审批点、保存检查点、向客户端发送中断事件、用户查看并提交决定、更新状态、从暂停位置继续。状态编辑还可以用于修正参数或从历史快照重新执行，但必须明确编辑者和分叉来源。

## 实践要点

- 为前端定义稳定事件包络，至少包含类型、Thread、节点、时间和负载。
- SSE 适合服务端单向推送；需要双向低延迟交互时再考虑 WebSocket。
- 审批界面应展示待执行动作、参数、影响范围和可选决定，而不是只显示“是否批准”。
- 恢复接口必须幂等，并防止同一中断被重复提交。
- 不要向普通用户暴露未经处理的内部推理；应输出可解释的状态和操作摘要。
- 长连接需要处理断线重连、事件重放、背压和超时。

## 决策依据

聊天产品通常以 `messages` 提供文本体验，并以 `events` 或 `custom` 补充工具和业务状态。审批型工作流应把 Checkpointer 视为必需基础设施，而不是可选增强；没有可靠持久化，中断后就无法安全恢复。

## 相关主题

- [[entities/langgraph|LangGraph]]
- [[concepts/langgraph-core-and-persistence|LangGraph 核心模型与持久化]]
- [[summaries/langgraph-implementation|LangGraph 实施方案]]
- [[concepts/langgraph-multi-agent|LangGraph 多 Agent 编排]]
