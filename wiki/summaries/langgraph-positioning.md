---
id: summary-langgraph-positioning
type: summary
title: "LangGraph 定位与选型"
status: draft
updated: 2026-08-14
owner: "llm-wiki-expert"
sources:
  - ../../raw/sources/docs/langgraph-technical-report/08-summary-and-comparison.md
---

# LangGraph 定位与选型

## 概览

LangGraph 的差异化能力是把状态机、有环控制流、检查点和人类在环组合为一个 Agent 运行时。它适合需要持续状态和精确控制的生产流程，不是所有 LLM 应用的默认答案；简单链、纯检索和大规模数据处理通常有更直接的工具。

## 核心优势

- 任意图拓扑支持循环、条件分支、并行和嵌套。
- State 与 Reducer 提供集中、可声明的数据更新模型。
- 检查点支持恢复、回放、时间旅行和审批流程。
- 流式协议可暴露消息、状态更新和工具事件。
- 可与 LangChain 模型工具生态和 LangSmith 可观测能力集成。

## 主要代价

- 开发者需要理解图、状态归约、线程和检查点语义。
- 循环与嵌套图的调试成本高于线性流程。
- 多层调度和持久化会增加运行开销。
- 与 LangChain 生态的深度结合可能增加迁移成本。
- 来源评估时 API 仍在演进，生产项目需要控制版本升级。

## 框架定位比较

| 方案 | 核心关注 | 更适合的任务 |
|---|---|---|
| LangGraph | 有状态图与可控执行 | 审批、循环工具调用、复杂 Agent 工作流 |
| AutoGen | Agent 间对话与群聊 | 多角色讨论和协作实验 |
| CrewAI | 角色化任务流水线 | 快速构建顺序明确的多 Agent 原型 |
| LlamaIndex | 数据连接、索引与检索 | 以 RAG 和知识问答为核心的系统 |
| Haystack | 检索与 Pipeline | 搜索、问答和文档处理管道 |
| LangChain LCEL | 线性组合 | 简单、确定的模型调用链 |

## 决策依据

当应用同时需要复杂分支、循环、持久化或人工审批时，LangGraph 的收益最明显。主要需求是多 Agent 自由讨论时可以优先评估 AutoGen；追求快速角色流水线时可评估 CrewAI；以 RAG 为主时应先选择 LlamaIndex 或 Haystack，再按控制流复杂度决定是否叠加 LangGraph。

不适合优先采用 LangGraph 的场景包括简单线性链、纯数据 ETL、只有一次模型调用的接口，以及对极低调度开销有硬要求的实时推理服务。

## 实践要点

- 用最小原型验证循环、恢复和审批三个关键能力，不要只比较示例代码长度。
- 评估状态存储、可观测性和运维成本，而不只评估编排 API。
- 将框架能力与业务控制流分离，避免在节点内直接依赖过多框架对象。
- 对竞品结论进行版本化复核，避免把特定时间点的功能矩阵当作长期事实。

## 相关主题

- [[entities/langgraph|LangGraph]]
- [[concepts/langgraph-core-and-persistence|LangGraph 核心模型与持久化]]
- [[concepts/langgraph-multi-agent|LangGraph 多 Agent 编排]]
- [[summaries/agent-cli-language-selection|Agent-CLI 开发语言选型]]
