---
id: entity-langgraph
type: entity
title: "LangGraph"
status: draft
updated: 2026-08-14
owner: "llm-wiki-expert"
sources:
  - ../../raw/sources/docs/langgraph-technical-report/README.md
  - ../../raw/sources/docs/langgraph-technical-report/01-overview-and-architecture.md
---

# LangGraph

## 概览

LangGraph 是 LangChain 团队面向 Agent 编排提供的图结构框架，同时支持 Python 与 TypeScript。它把 Agent 建模为有状态、可循环的计算图，重点解决复杂控制流、状态合并、执行持久化、流式输出和人类干预，而不是替代底层模型调用库。

## 核心定位

LangGraph 的关键判断是“Agent 是有状态的循环图，而不是线性链”。节点承载具体计算，边表达控制流，State 作为节点间的数据契约，Pregel 风格执行引擎按轮次调度可执行节点并合并状态更新。

| 组件 | 职责 |
|---|---|
| `StateGraph` | 声明状态模型、节点和边 |
| Node | 执行模型、工具或业务逻辑，返回局部状态更新 |
| Edge Router | 根据固定规则或运行时状态决定下一步 |
| Pregel Engine | 调度节点、处理并行轮次并推进图执行 |
| Checkpointer | 保存线程内检查点，支持恢复和回溯 |
| Store | 保存跨线程共享的数据 |
| Stream | 输出状态、增量、消息或自定义事件 |

## 主要能力

- 支持循环、条件分支、并行节点和嵌套子图。
- 通过 Reducer 为不同 State 字段定义合并策略。
- 通过检查点保存执行历史，支撑会话恢复、人类在环和时间旅行。
- 通过多种流式模式向客户端暴露结果、增量和内部事件。
- 可以与 LangChain 的模型、工具以及 LangSmith 可观测能力组合使用。

## 与线性链的边界

简单的固定顺序调用使用线性链更直接；当流程需要重试循环、动态路由、审批暂停或多 Agent 协作时，图模型才体现价值。引入 LangGraph 会增加状态设计、图调试和运行时治理成本，不应把所有 LLM 调用都包装成复杂图。

## 实践要点

- 先定义 State 契约，再设计节点，避免节点通过隐式副作用交换数据。
- 节点保持单一职责，只返回需要更新的字段。
- 循环必须具备终止条件、重试上限和可观测事件。
- 生产环境应在设计阶段同时考虑检查点后端、线程标识和流式协议。

## 相关主题

- [[concepts/langgraph-core-and-persistence|LangGraph 核心模型与持久化]]
- [[summaries/langgraph-interaction|LangGraph 流式交互与人类在环]]
- [[concepts/langgraph-multi-agent|LangGraph 多 Agent 编排]]
- [[summaries/langgraph-implementation|LangGraph 实施方案]]
- [[summaries/langgraph-positioning|LangGraph 定位与选型]]
