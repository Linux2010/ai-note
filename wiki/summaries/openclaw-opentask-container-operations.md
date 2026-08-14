---
id: summary-openclaw-opentask-container-operations
type: source-summary
title: "OpenClaw 中的 OpenTask 容器运维"
status: draft
updated: 2026-08-14
owner: "llm-wiki-expert"
sources:
  - ../../raw/sources/openclaw/OpenTask-容器配置指南.md
---

# OpenClaw 中的 OpenTask 容器运维

## 概览

OpenTask 在 OpenClaw 容器中的职责是提供可轮询、可声明状态的任务队列。可靠运行依赖三个闭环：容器能够访问宿主机 API，任务状态严格按 `pending → start → complete/fail` 转换，定时执行结果能够通过固定 Channel 送达用户。

## 核心内容与机制

### 容器到宿主机的寻址

容器内的 `127.0.0.1` 指向容器自身，因此宿主机上的 OpenTask API 通常应通过 `host.docker.internal` 访问。运行时至少需要配置服务地址、API Key 和 Bot 名称；Bot 名称用于筛选分配给当前执行者的待处理任务。

### 任务状态机

标准处理顺序是：

1. 查询分配给当前 Bot 的待执行任务。
2. 按 `P0 > P1 > P2` 选择优先级最高的任务。
3. 在真正执行前调用 `start`，使外部系统看到任务已被领取。
4. 成功后调用 `complete`；无法完成时调用 `fail`，不能让任务长期停留在运行态。

这一状态机把“Agent 说自己完成了”与“调度系统记录完成”区分开。两者必须一致，任务生命周期才算闭合。

### Cron 与结果送达

定时检查适合使用隔离会话和 `agentTurn` payload，避免依赖某个交互会话仍然存活。送达配置应显式指定 `channel` 和接收者，不使用依赖最近会话的模糊目标。原文特别记录：输出包含 `HEARTBEAT_OK` 时可能被系统识别为纯心跳确认，从而抑制报告送达；OpenTask 检查报告应使用普通状态摘要。

## 实践要点与决策依据

- 把 API Key 作为环境变量注入，不写入任务消息或脚本输出。
- API 请求应禁用不必要的代理继承，并分别验证宿主机和容器内的连通性。
- 使用 `sessionTarget: isolated`，并为任务轮询设置有限超时，防止定时任务占用会话。
- 即使没有待执行任务，也输出简短、可送达的检查结果，但不要伪装成任务完成事件。
- 对每次领取记录任务 ID；异常路径必须调用失败接口或交由恢复流程处理。
- `host.docker.internal` 在部分 Docker 网络环境中可能不可解析，遇到问题应转入容器网络诊断，而不是先修改任务逻辑。

## 相关主题

- [[summaries/openclaw-installation-and-runtime-management|OpenClaw 安装与运行时管理]]
- [[summaries/openclaw-telegram-connectivity-troubleshooting|OpenClaw Telegram 连通性排障]]
- [[summaries/openclaw-network-and-remote-access|OpenClaw 网络与远程访问]]
