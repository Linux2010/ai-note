---
id: entity-hermes-agent
type: entity
title: "Hermes Agent"
status: draft
updated: 2026-08-14
owner: "llm-wiki-expert"
sources:
  - ../../raw/sources/hermes-agent/overview.md
---

# Hermes Agent

## 概览

Hermes Agent 是 Nous Research 开发的 Python Agent 项目。来源基于核心源码和 PR 审查，将其定位为具备自我改进闭环的长期运行系统，而非单纯聊天机器人。它把技能演化、持久化记忆、多平台消息网关、模型适配和多种执行后端组合到同一产品中。

## 核心机制

### 自我改进闭环

Hermes 可以把复杂任务的操作步骤沉淀为可复用 Skill，并在后续使用遇到问题时改进指令。持久化记忆、SQLite FTS5 会话搜索和用户建模为这一闭环提供历史上下文。其核心价值不是“自动生成更多文件”，而是让经验能够被检索、修订并再次参与执行。

### 多平台网关

单一网关面向 Telegram、Discord、Slack、WhatsApp、Signal 和 Email 分发消息，并追求跨平台对话连续性。该模式把 Agent 核心与渠道适配分离，使调度、记忆和工具能力可以被多个入口复用。

### 执行与模型抽象

Hermes 支持本地、Docker、SSH、云端和 HPC 等终端后端，并通过模型适配层连接多种服务商。子代理和 Python RPC 用于并行任务或低上下文成本的流水线，MCP 则提供外部工具扩展接口。

## 架构要点

| 模块 | 职责 |
|---|---|
| Agent 主循环 | 对话、工具调用和执行协调 |
| 工具编排 | 工具发现、参数调用和结果处理 |
| SessionDB | SQLite 会话持久化与 FTS5 搜索 |
| Context Compressor | 长上下文自动压缩 |
| 模型适配器 | 处理不同模型协议和特殊内容块 |
| Gateway | 渠道消息接入、命令和分发 |
| Cron | 内置定时任务与结果投递 |
| ACP Adapter | 编辑器和外部客户端集成 |

## 实践与评估

Hermes 适合需要长期运行、跨渠道访问、模型自由和经验沉淀的个人或研究型 Agent。采用前应重点评估主循环单文件过大、API 快速变化、代码与文档不同步等维护风险，并用实际版本验证来源中记录的模型数量、测试规模和平台支持。

## 限制与边界

来源结论对应 2026 年 4 月的源码快照。社区活跃度、版本号、已修复问题和模型覆盖属于易变信息，本页保留其架构意义，但不把这些数值视为当前实时状态。

## 相关主题

- [[concepts/agent-long-term-memory|Agent 长期记忆分层架构]]
- [[concepts/langgraph-multi-agent|LangGraph 多 Agent 编排]]
- [[summaries/agent-cli-language-selection|Agent-CLI 开发语言选型]]
- [[summaries/claude-code-telegram-remote|Claude Code Telegram 远程通信]]
