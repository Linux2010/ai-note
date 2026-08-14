---
id: summary-openhorse-configuration
type: summary
title: "OpenHorse 配置与最佳实践"
status: draft
updated: 2026-08-14
owner: "llm-wiki-expert"
sources:
  - ../../raw/sources/openhorse/best-practices/openhorse-configuration-best-practices.md
---

# OpenHorse 配置与最佳实践

## 概览

OpenHorse 是 TypeScript 实现的 CLI Agent 框架，围绕工具编排、多模型、上下文压缩、MCP、分层记忆和权限控制组织能力。配置工作的核心不是填满参数，而是建立模型可用性、工具最小权限、记忆生命周期和故障降级之间的明确边界。

## 核心配置机制

用户配置主要提供 API 凭据、兼容端点、默认模型和备用模型；输出 Token、温度和重试等运行参数由 Agent 内部管理。配置来源存在文件、环境变量和运行时命令等层级，部署时应明确优先级并避免同一字段在多个位置长期漂移。

模型选择应基于任务类型、上下文长度、成本和服务可用性。备用模型用于主模型失败时降级，但必须验证两者的上下文窗口、工具调用协议和输出能力是否兼容。

## MCP 与权限

MCP 服务器扩展文件系统、Telegram、GitHub 和数据库等能力。每个服务器只应获得必要目录和资源，Token 与密码通过环境变量注入。连接层可使用心跳和指数退避恢复，但自动重连不能替代状态监控和错误提示。

工具确认模式决定操作风险：开发环境可以提高自动化程度，团队和生产环境应优先确认敏感调用。Bash 安全检查、文件范围限制和审计日志共同构成边界，任何单项检查都不能独立保证安全。

## 记忆与上下文

OpenHorse 区分工作记忆、会话记忆、长期记忆和语义搜索，并以用户、反馈、项目和参考资料分类。上下文接近容量上限时会自动压缩；重要决定应提前进入长期记忆，避免仅依赖对话摘要。

## 实践要点

- API Key 放入环境变量或密钥服务，不写入仓库配置。
- MCP 文件系统能力只开放任务所需目录，并定期检查服务器状态。
- 生产环境使用确认模式，保留工具调用审计记录。
- 长任务主动观察上下文占用，必要时拆分会话或执行压缩。
- 记忆按类型保存并定期清理、重建索引，避免过时信息持续召回。
- 为主模型和备用模型设计可验证的降级路径，不只配置模型名称。

## 排错顺序

Web 搜索失败时依次检查凭据、网络和提供商；MCP 失败时检查状态、命令路径和依赖；压缩过频时检查模型窗口和会话粒度；模型切换后出现上下文变化时，应确认新模型容量是否触发自动压缩。

## 限制与边界

来源对应 OpenHorse v0.1.21，模型名称、默认值、上下文窗口和命令可能随版本变化。实施时应以当前安装版本的配置 schema 和帮助信息为准。

## 相关主题

- [[summaries/agent-cli-language-selection|Agent-CLI 开发语言选型]]
- [[concepts/agent-long-term-memory|Agent 长期记忆分层架构]]
- [[summaries/openclaw-memory-search|OpenClaw Memory Search]]
- [[summaries/claude-code-telegram-remote|Claude Code Telegram 远程通信]]
