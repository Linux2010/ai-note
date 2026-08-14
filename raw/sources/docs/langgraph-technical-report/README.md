# LangGraph 完整技术方案报告

> 生成日期: 2026-04-26
> LangGraph 版本: 0.2.x+
> 适用语言: Python / TypeScript

## 目录

| # | 文档 | 内容概要 |
|---|------|----------|
| 1 | [概述与架构](./01-overview-and-architecture.md) | LangGraph 定位、核心特性、整体架构图、与 LangChain 关系 |
| 2 | [核心概念详解](./02-core-concepts.md) | State、StateGraph、节点、边、条件边、Entry/Exit 节点 |
| 3 | [状态持久化与检查点](./03-persistence-and-checkpoints.md) | Checkpointer 系统、Thread 管理、时间旅行、后端选择 |
| 4 | [流式输出与实时交互](./04-streaming.md) | 5 种 stream_mode、事件协议、SSE/WebSocket 集成 |
| 5 | [人类在环](./05-human-in-the-loop.md) | 中断/恢复、预检点、编辑状态、审核批准 |
| 6 | [多 Agent 系统](./06-multi-agent.md) | Supervisor、Pipeline、 Hierarchical、手写协作 |
| 7 | [完整技术方案与示例](./07-technical-plan-and-examples.md) | 端到端示例、生产架构、性能优化、安全规范 |
| 8 | [总结与竞品对比](./08-summary-and-comparison.md) | 技术总结、竞品对比、选型决策树、未来趋势 |

## 快速开始

如果你只想了解核心：
1. 先看 [01-overview-and-architecture](./01-overview-and-architecture.md) 理解全局
2. 再看 [07-technical-plan-and-examples](./07-technical-plan-and-examples.md) 获取完整示例
3. 按需查阅其他章节深入细节

## 报告定位

本报告面向 **准备在项目中采用 LangGraph 构建 Agent 应用的开发者和架构师**，涵盖：
- **技术原理**: 图执行引擎、状态归约、检查点机制
- **实践指南**: 完整示例代码、生产架构设计
- **选型决策**: 与主流 Agent 框架的客观对比
