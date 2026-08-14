---
id: summary-openclaw-group-chat-and-telegram-routing
type: source-summary
title: "OpenClaw 群聊行为与 Telegram 多 Agent 路由"
status: draft
updated: 2026-08-14
owner: "llm-wiki-expert"
sources:
  - ../../raw/sources/openclaw/best-practices/group-chat-configuration-best-practices.md
  - ../../raw/sources/openclaw/best-practices/multi-agent-telegram.md
  - ../../raw/sources/openclaw/solutions/telegram-multi-agent-routing.md
---

# OpenClaw 群聊行为与 Telegram 多 Agent 路由

## 概览

群聊中的 Agent 同时受技术路由和社交边界约束。技术上，需要把 Telegram Bot 账号、群组 peer 和 Agent binding 对齐；行为上，应默认克制，只在被提及、明确被请求或确有增量价值时参与，且不能把私聊信息带入群组。

## 核心内容与机制

### Telegram 平台边界

Telegram Bot 通常无法像人类用户一样消费其他 Bot 的消息，OpenClaw 的出站消息也不会自动作为新的入站事件再次路由。因此，让 Bot 在群里彼此 `@` 并形成稳定协作链并不可靠。原文推荐由用户直接提及目标 Bot，或者只保留一个协调 Bot，由其在内部调用专业 Agent。

### 路由优先级

原文总结的匹配顺序是：精确 `peer` 规则优先，其次是同一 Channel 下的 `accountId`，然后是 Channel 级规则，最后回退到默认 `main` Agent。只配置带群组 peer 的 binding，会导致私聊无法匹配专用 Agent，最终落到 main。

每个 Telegram 账号通常需要两类规则：带群组 peer 的规则负责指定群内路由，不带 peer 的 accountId 规则负责该账号的私聊和其他消息。`bindings[].match.accountId` 必须与 `channels.telegram.accounts` 的键完全一致。

### 群聊参与策略

群聊的默认模式应是 `mention-only` 或受控的智能判断。Agent 应避免回复每条消息、重复发送多个反应或替主人做决定。表情反应可用于轻量确认，但要限制频率。涉及日程、联系人、邮件和其他私人上下文时，群聊输出必须采用更严格的隐私边界。

## 实践要点与决策依据

- 多 Agent Telegram 的稳妥方案是“一账号一 Bot、一 accountId 一 Agent”，用户名避免包含关系，降低 mention 误匹配。
- 群组使用精确 peer 绑定和 `requireMention`；私聊增加 accountId 级绑定，避免全部回退到 main。
- BotFather 的 Privacy Mode 是否需要关闭取决于所需群组消息范围；开放后应同步收紧 OpenClaw 的访问控制。
- 群聊中不暴露私聊记忆，不替用户承诺，不在问题已经回答时继续刷屏。
- 配置完成后应分别设计“每个 Bot 私聊”“群组逐个提及”“无提及保持静默”三类验收场景。
- 文档中的若干群聊字段是实践性示例，可能不是所有版本的正式 schema；应将行为原则和实际配置字段分开验证。

## 相关主题

- [[summaries/openclaw-discord-operations|OpenClaw Discord Bot 配置与运维]]
- [[summaries/openclaw-telegram-connectivity-troubleshooting|OpenClaw Telegram 连通性排障]]
- [[summaries/openclaw-runtime-configuration|OpenClaw 运行时配置体系]]
