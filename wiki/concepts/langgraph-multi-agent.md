---
id: concept-langgraph-multi-agent
type: concept
title: "LangGraph 多 Agent 编排"
status: draft
updated: 2026-08-14
owner: "llm-wiki-expert"
sources:
  - ../../raw/sources/docs/langgraph-technical-report/06-multi-agent.md
---

# LangGraph 多 Agent 编排

## 概览

多 Agent 系统通过角色分工降低单个提示词和工具集合的复杂度。LangGraph 将每个 Agent 表示为节点或子图，并通过共享状态、消息传递和路由规则组织协作。是否采用多 Agent，取决于任务能否形成清晰边界，而不是角色数量是否足够多。

## 核心模式

| 模式 | 控制方式 | 适用场景 | 主要风险 |
|---|---|---|---|
| Pipeline | 固定顺序传递结果 | 步骤稳定的加工流程 | 前序错误逐级放大 |
| Routing | 路由器选择专业 Agent | 输入类型明确、处理器互斥 | 错误分类导致错误分派 |
| Parallel | 多个 Agent 同时执行后合并 | 独立分析、交叉验证 | 合并冲突与资源峰值 |
| Supervisor | 监督者动态选择 Worker | 开放式复杂任务 | 调度循环和成本失控 |
| Subgraph | 子图作为上层节点 | 层级团队和复用流程 | 状态映射与调试复杂 |

## 通信机制

共享状态模式把所有 Agent 的数据放入同一 State，简单直接但容易造成字段膨胀和耦合。消息传递模式只交换显式消息，边界清晰但需要更多协议和调度代码。混合模式通常最实用：共享少量全局任务信息，详细过程保留在各子图内部。

Reducer 是并行协作的关键。当多个 Agent 同时写入结果时，合并规则必须可预测；列表追加、按 Agent 命名的结果映射或显式聚合节点都比“最后写入者获胜”更安全。

## 选型依据

- 步骤固定且依赖明确时选择 Pipeline。
- 专业能力互斥且分类可靠时选择 Routing。
- 子任务彼此独立且延迟重要时选择 Parallel。
- 下一步依赖动态判断时选择 Supervisor。
- 一个领域流程需要复用或独立演进时封装为 Subgraph。

## 实践要点

- 为每个 Agent 定义输入、输出、工具权限和终止条件。
- Supervisor 必须设置最大轮次、预算和无进展检测。
- 并行节点只共享必要状态，避免把完整对话复制给所有角色。
- 聚合节点应保留来源 Agent 和冲突信息，不要无痕覆盖不同结论。
- 使用子图隔离局部状态，并显式定义与父图的字段映射。
- 优先证明单 Agent 无法稳定完成任务，再引入多 Agent 复杂度。

## 相关主题

- [[entities/langgraph|LangGraph]]
- [[concepts/langgraph-core-and-persistence|LangGraph 核心模型与持久化]]
- [[summaries/langgraph-interaction|LangGraph 流式交互与人类在环]]
- [[summaries/langgraph-implementation|LangGraph 实施方案]]
