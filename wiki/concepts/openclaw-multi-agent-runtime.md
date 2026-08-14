---
id: concept-openclaw-multi-agent-runtime
type: concept
title: "OpenClaw 多代理运行时"
status: draft
updated: 2026-08-14
owner: "llm-wiki-expert"
sources:
  - ../../raw/sources/openclaw/architecture/multi-agent.md
  - ../../raw/sources/openclaw/best-practices/sessions-spawn-async-monitoring.md
---

# OpenClaw 多代理运行时

## 概览

OpenClaw 多代理运行时把“角色拓扑”和“异步执行”作为两个独立但协作的层次：主控代理负责接收、分派和汇总任务，领域代理负责受限范围内的工作；子任务通过独立会话异步执行，避免主会话被长任务阻塞。[多代理架构](../../openclaw/architecture/multi-agent.md) [异步任务监控](../../openclaw/best-practices/sessions-spawn-async-monitoring.md)

## 核心机制

### 角色与路由

架构来源将核心代理定义为协调入口，并把专业能力分配给领域代理。路由规则应以职责、通道和身份边界为依据，而不是让所有代理共享同一角色与上下文。这样可以减少任务串扰，并使每个代理的权限和数据范围保持可解释。[多代理架构](../../openclaw/architecture/multi-agent.md)

### 执行隔离

来源提供了容器隔离和原生多进程两类部署思路。容器方案更强调进程与文件边界，原生方案更接近宿主机能力但隔离较弱。部署隔离与子会话隔离不能混为一谈：多个子会话可以共享 Gateway 控制面，而各自保持独立任务上下文。[多代理架构](../../openclaw/architecture/multi-agent.md) [异步任务监控](../../openclaw/best-practices/sessions-spawn-async-monitoring.md)

### 异步子会话

`sessions_spawn` 用于提交非阻塞任务，调用方获得运行标识和子会话标识后继续当前工作。完成结果优先通过通知机制返回主会话，而不是持续轮询。必要时可通过会话列表和历史记录补充查看状态与执行证据。[异步任务监控](../../openclaw/best-practices/sessions-spawn-async-monitoring.md)

### 权限与资源治理

子代理并不天然拥有主代理的全部能力。运行时需要明确会话深度、可用工具、并发上限、超时和清理策略，以防止递归派生、资源悬挂或权限扩散。[异步任务监控](../../openclaw/best-practices/sessions-spawn-async-monitoring.md)

## 实践要点

- 先定义主控代理和领域代理的职责，再设计通道路由和消息协议。
- 安全优先时采用更强执行隔离；依赖宿主机原生能力时，显式接受隔离减弱的代价。
- 默认采用异步提交与完成通知，只有在诊断或通知缺失时才查询会话历史。
- 对每类子任务设置超时、并发和清理策略，并保留运行标识以支持追踪。
- Gateway 重启可能影响待发送通知，因此重要任务不能只依赖一次完成推送。[异步任务监控](../../openclaw/best-practices/sessions-spawn-async-monitoring.md)

## 证据与来源

- [OpenClaw 多代理架构完整方案](../../openclaw/architecture/multi-agent.md)：支持角色划分、通信方式以及容器与原生部署取舍。
- [OpenClaw 异步任务监控方案](../../openclaw/best-practices/sessions-spawn-async-monitoring.md)：支持子会话语义、通知模式、权限边界和并发治理。

## 冲突与限制

架构来源强调代理实例之间的部署隔离，异步来源则描述共享 Gateway 下的子会话执行。两者对应不同层级，不能据此声称所有子代理均具有物理隔离。通知机制也不是持久任务队列，关键任务仍需额外的状态记录和恢复路径。

## 相关主题

- [OpenClaw 架构总览](../summaries/openclaw-architecture-overview.md)
- [OpenClaw 记忆架构](../summaries/openclaw-memory-architecture.md)
- [OpenClaw Memory Search 运维](../summaries/openclaw-memory-search-operations.md)
