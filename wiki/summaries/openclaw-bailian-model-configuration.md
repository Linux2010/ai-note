---
id: summary-openclaw-bailian-model-configuration
type: source-summary
title: "OpenClaw 百炼模型配置"
status: draft
updated: 2026-08-14
owner: "llm-wiki-expert"
sources:
  - ../../raw/sources/openclaw/best-practices/bailian-coding-plan-complete-guide.md
  - ../../raw/sources/openclaw/best-practices/bailian-model-configuration.md
  - ../../raw/sources/openclaw/best-practices/bailian-models-best-practices.md
  - ../../raw/sources/openclaw/claude-code-bailian-coding-plan.md
---

# OpenClaw 百炼模型配置

## 概览

在 OpenClaw 中接入阿里云百炼 Coding Plan，需要同时完成 Provider 定义、模型目录注册、默认模型选择和凭证注入。关键不是堆叠模型条目，而是保证端点、协议、模型能力声明和 Agent 可选列表彼此一致。

本页以 OpenClaw 为主，Claude Code 的 Anthropic 兼容配置仅用于说明边界：两者使用的配置文件、环境变量和兼容端点不同，不能直接复制。

## 核心内容与机制

### Provider 与模型注册

百炼 Provider 应定义在 `openclaw.json` 的 `models.providers.bailian` 下。Coding Plan 使用专属端点，原文配置采用 `openai-completions` 兼容协议。每个模型条目描述 ID、输入类型、上下文窗口和最大输出等能力。

Provider 中声明模型后，还要在 `agents.defaults.models` 中加入对应的 `bailian/<model-id>` 条目，否则模型可能已配置却不出现在可选目录中。`agents.defaults.model` 或具体 `agents.list[].model` 再决定默认模型和 Agent 级覆盖。

### 模型分工

原文按任务建议将通用对话、复杂推理、代码生成、代码审查、长文档和多模态分配给不同模型。更稳妥的决策顺序是先看输入模态和上下文需求，再看延迟、输出上限和套餐权限，最后决定默认与备用模型。

### 配置冲突与版本边界

不同来源对 `MiniMax-M2.5` 的上下文和最大输出参数存在冲突：较早文档给出 `204800/131072`，较新的完整指南给出 `196608/32768`。这类参数不应由 wiki 擅自裁决；部署时应以当前百炼接口、套餐说明和实际模型列表为准。Thinking 兼容字段、支持模型数量和套餐范围同样属于时效性信息。

## 实践要点与决策依据

- Provider 放入 `openclaw.json`，不要依赖旧版独立 `models.json` 作为主配置来源。
- API Key 使用环境变量、SecretRef 或认证配置注入，避免直接进入版本控制。
- Provider 模型列表和 `agents.defaults.models` 保持一一对应，并为常用模型设置稳定别名。
- 多容器可共享模型策略，但默认模型应按职责分配；代码 Agent 和分析 Agent 不必使用同一模型。
- 任何模型参数变更都应连同来源日期记录，避免旧配置覆盖新能力边界。
- Claude Code 使用 Anthropic 兼容端点和 `~/.claude/settings.json`；该方案不能替代 OpenClaw Provider 配置。

## 相关主题

- [[summaries/openclaw-runtime-configuration|OpenClaw 运行时配置体系]]
- [[summaries/openclaw-installation-and-runtime-management|OpenClaw 安装与运行时管理]]
- [[summaries/openclaw-network-and-remote-access|OpenClaw 网络与远程访问]]
