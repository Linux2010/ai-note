---
id: concept-langgraph-core-and-persistence
type: concept
title: "LangGraph 核心模型与持久化"
status: draft
updated: 2026-08-14
owner: "llm-wiki-expert"
sources:
  - ../../raw/sources/docs/langgraph-technical-report/02-core-concepts.md
  - ../../raw/sources/docs/langgraph-technical-report/03-persistence-and-checkpoints.md
---

# LangGraph 核心模型与持久化

## 概览

LangGraph 的运行模型由 State、Node、Edge 和 Checkpoint 共同构成。State 定义数据契约，Node 产生局部更新，Edge 决定控制流，Checkpointer 将每一步状态保存为可恢复历史。理解这四者的边界，是正确构建可循环、可暂停 Agent 的基础。

## State 与 Reducer

State 是图内所有节点共享的数据载体。节点不应直接修改传入对象，而应返回局部更新；每个字段如何合并由 Reducer 决定。消息列表可以按消息 ID 合并，普通列表可以追加，字典可以采用自定义合并逻辑，没有 Reducer 的字段通常采用覆盖语义。

State schema 同时承担类型约束和协作协议。字段应按业务含义命名，并区分输入、过程状态、结果和错误信息。把密钥或密码写入 State 会使其进入检查点，因此敏感运行参数应通过配置注入。

## Node 与 Edge

Node 是接受 State 并返回更新的可调用单元，适合封装模型调用、工具执行或确定性业务逻辑。Edge 表达节点间转移，包括固定边、条件边、入口、出口和由同一上游触发的并行分支。

节点设计应保持单一职责并显式处理预期错误。条件路由应返回稳定、可枚举的目标；循环边必须具备终止条件。多个并行节点更新同一字段时，必须预先定义可结合、可预测的 Reducer。

## Checkpointer 与 Thread

Checkpointer 在执行过程中保存 State 快照、元数据和待处理信息。`thread_id` 将连续调用归入同一执行历史，使对话记忆、崩溃恢复、人类审批和历史回放成为可能。时间旅行本质上是从某个历史检查点恢复或分叉执行，而不是随意修改当前内存。

| 后端 | 适用场景 | 主要限制 |
|---|---|---|
| MemorySaver | 本地开发与调试 | 进程退出后丢失，不能跨进程共享 |
| SqliteSaver | 个人应用与原型 | 并发写入能力有限 |
| PostgresSaver | 单节点或集群生产环境 | 需要数据库运维和连接管理 |
| RedisSaver | 高并发、低延迟场景 | 需要额外持久化与一致性策略 |

## Checkpointer 与 Store 的区别

Checkpointer 保存单个 Thread 的执行快照，回答“这个流程执行到了哪里”；Store 保存跨 Thread 共享信息，回答“不同会话共同知道什么”。用户偏好和长期知识更适合 Store，节点恢复和审批状态更适合 Checkpointer。

## 实践要点

- 在图编译前确定 `thread_id` 生成、租户隔离和生命周期策略。
- 对 State 字段设置大小上限，避免消息和工具结果无限膨胀。
- 生产环境使用支持并发和事务的后端，并配置连接池、备份与清理策略。
- 将错误状态显式写入 State，便于路由到重试、人工处理或终止节点。
- 时间旅行和状态编辑应留下审计记录，避免历史分叉无法解释。

## 相关主题

- [[entities/langgraph|LangGraph]]
- [[summaries/langgraph-interaction|LangGraph 流式交互与人类在环]]
- [[concepts/langgraph-multi-agent|LangGraph 多 Agent 编排]]
- [[concepts/agent-long-term-memory|Agent 长期记忆分层架构]]
