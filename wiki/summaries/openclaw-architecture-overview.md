---
id: summary-openclaw-architecture-overview
type: summary
title: "OpenClaw 架构总览"
status: draft
updated: 2026-08-14
owner: "llm-wiki-expert"
sources:
  - ../../raw/sources/openclaw/architecture/core-architecture.md
  - ../../raw/sources/openclaw/architecture/openclaw-architecture-and-clone-guide.md
---

# OpenClaw 架构总览

## 概览

OpenClaw 是一个以 Gateway 为控制入口、以 Agent 循环为执行核心、以插件和会话为扩展边界的事件驱动系统。两份架构来源分别从源码调用链和复刻视角描述这一结构：前者强调消息如何穿过 WebSocket、路由、Handler 和 Agent 执行层，后者补充客户端状态、插件槽位、会话持久化和部署边界。[核心架构](../../openclaw/architecture/core-architecture.md) [架构与复刻指南](../../openclaw/architecture/openclaw-architecture-and-clone-guide.md)

## 核心内容与机制

### 网关与请求链路

系统入口由 HTTP/WebSocket Gateway 承担。客户端完成连接与认证后，以带有类型、方法、请求标识和参数的消息帧发起调用；网关先执行协议校验和权限判断，再把请求分发给对应 Handler。[核心架构](../../openclaw/architecture/core-architecture.md)

这条主链可概括为：服务器启动、WebSocket 消息处理、请求路由、业务 Handler、Agent 执行。分层的价值不在目录形式，而在于把连接管理、控制策略和业务执行分离，使通道接入、配置管理、定时任务与 Agent 调用可以共享统一入口。[核心架构](../../openclaw/architecture/core-architecture.md)

### Agent 与工具循环

Agent 层负责组装会话上下文并调用模型。模型返回文本时向客户端发送结果；返回工具调用时，运行时执行工具，把结果追加回上下文，再次请求模型，直到满足终止条件。会话转录随后进入持久化流程。这一“模型判断、工具执行、结果回填”的闭环，是 OpenClaw 从对话系统转为任务执行系统的关键机制。[核心架构](../../openclaw/architecture/core-architecture.md)

### 会话、插件与后台服务

会话层保存连续交互所需的历史和状态，插件层则通过能力槽位与具体实现解耦可替换组件，例如记忆、浏览器或搜索能力。运行时还承载通道健康检查、定时任务和其他持续服务，因此 Gateway 同时是请求控制面与长期运行服务的协调点。[架构与复刻指南](../../openclaw/architecture/openclaw-architecture-and-clone-guide.md)

## 实践要点

- 先稳定网关协议、认证和路由边界，再扩展 Handler 与 Agent 工具，避免能力直接绕过控制面。
- 工具执行必须把结果回填给模型并保留会话证据，不能把一次工具调用误当成完整任务闭环。
- 插件能力应通过稳定槽位接入，减少对 Gateway 和 Agent 主循环的硬编码侵入。
- 复刻指南中的示例实现用于说明结构，不等同于当前 OpenClaw 源码；落地时应重新核对版本和实际模块。[架构与复刻指南](../../openclaw/architecture/openclaw-architecture-and-clone-guide.md)

## 证据与来源

- [OpenClaw 核心架构与消息处理循环](../../openclaw/architecture/core-architecture.md)：支持五层调用链、网关协议、Handler 分发和工具循环。
- [OpenClaw 架构设计与复刻指南](../../openclaw/architecture/openclaw-architecture-and-clone-guide.md)：支持客户端状态、插件模型、会话机制、部署结构和复刻边界。

## 冲突与限制

两份来源存在大量重叠，但用途不同：核心架构文档更接近实现说明，复刻指南包含抽象、示例代码和替代实现建议。后者不能单独证明当前源码行为；其中涉及数量、模块名称和运行参数的内容也可能随版本变化。

## 相关主题

- [OpenClaw 多代理运行时](../concepts/openclaw-multi-agent-runtime.md)
- [OpenClaw 记忆架构](openclaw-memory-architecture.md)
- [OpenClaw Memory Search 运维](openclaw-memory-search-operations.md)
