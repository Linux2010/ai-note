---
id: summary-claude-code-telegram-remote
type: summary
title: "Claude Code Telegram 远程通信"
status: draft
updated: 2026-08-14
owner: "llm-wiki-expert"
sources:
  - ../../raw/sources/claude-code-telegram-remote-communication.md
---

# Claude Code Telegram 远程通信

## 概览

来源将 Claude Code 的 Telegram 远程通信描述为 MCP Server 与 Telegram Bot 的组合，而不是由普通 Skill 直接承载后台通信。MCP 层向 Agent 暴露回复、表情、编辑和附件下载工具；Skills 负责访问控制和配置等用户主动操作。

## 核心架构

Telegram Bot 通过轮询接收入站消息，将文本、图片和元数据转换为渠道消息通知。Claude Code Session 处理消息后调用 MCP 工具完成出站回复。渠道状态目录保存 Token、访问控制、下载文件和进程信息，因此需要与普通工作文件隔离。

| 组件 | 职责 |
|---|---|
| MCP Server | 注册渠道工具并传递消息通知 |
| Telegram Bot | 接收更新、发送消息和处理交互按钮 |
| State Directory | 保存凭据、授权、附件和 PID 等状态 |
| Access Skill | 配对、白名单、群组和确认反应管理 |
| Configure Skill | 写入 Bot Token 与基础配置 |
| Permission Relay | 把敏感工具审批发送到 Telegram 并回传决定 |

## 访问控制机制

私聊可以采用配对、白名单或禁用策略。默认配对流程向未知用户返回短码，再由本地终端批准；白名单策略对未知用户静默拒绝。群组触发可以要求提及 Bot、回复 Bot，或限制特定成员。

权限转发使远程用户可以处理 Claude Code 的敏感操作请求，但它同时扩大了攻击面。渠道消息不能获得与本地终端输入同等的 Skill 调用权限，状态文件也不应通过发送附件工具泄露。

## 多 Bot 方案

来源推荐为每个 Bot 使用独立进程和独立状态目录。这样可以隔离 Token、访问控制、PID 和 Claude Session，适合不同用户或用途；代价是每个实例独立占用资源和模型额度。同一 Telegram Token 不能被多个轮询实例同时使用。

## 实践要点

- 默认采用配对或白名单，不向公网开放无约束私聊。
- 为不同用途的 Bot 分离 Token、状态目录、进程和权限范围。
- 权限消息展示操作内容和影响范围，并设置过期时间与防重放标识。
- 附件下载后执行类型、大小和存储路径校验。
- 监控 PID、轮询冲突和网络重连，避免重复消费或孤儿进程。
- 把渠道消息视为不可信输入，禁止其绕过本地命令和 Skill 的安全边界。

## 限制与边界

来源基于特定插件源码与本地安装结构，目录名、工具 schema 和启动方式可能随版本调整。正式部署前应以当前插件清单和 MCP 配置为准。

## 相关主题

- [[entities/hermes-agent|Hermes Agent]]
- [[summaries/openhorse-configuration|OpenHorse 配置与最佳实践]]
- [[concepts/tool-call-protocol|工具调用协议]]
