# 已录入来源目录

本目录记录主题化知识页与集中来源的对应关系。表中来源路径均相对于 `raw/`，并镜像迁移前的仓库路径；原文位于 `sources/**`，知识页位于 `../wiki/**`。

## Agent CLI / 记忆

| 聚合专题 | 来源集合 |
|---|---|
| [Agent CLI 语言选型](../wiki/summaries/agent-cli-language-selection.md) | `sources/docs/agent-cli-language-comparison.md`<br>`sources/docs/python-agent-cli-research.md`<br>`sources/research/java-agent-cli-feasibility.md` |
| [Agent 长期记忆](../wiki/concepts/agent-long-term-memory.md) | `sources/docs/elite-longterm-memory-guide.md` |

## LangGraph

| 聚合专题 | 来源集合 |
|---|---|
| [LangGraph](../wiki/entities/langgraph.md) | `sources/docs/langgraph-technical-report/README.md`<br>`sources/docs/langgraph-technical-report/01-overview-and-architecture.md` |
| [LangGraph 核心机制与持久化](../wiki/concepts/langgraph-core-and-persistence.md) | `sources/docs/langgraph-technical-report/02-core-concepts.md`<br>`sources/docs/langgraph-technical-report/03-persistence-and-checkpoints.md` |
| [LangGraph 交互模式](../wiki/summaries/langgraph-interaction.md) | `sources/docs/langgraph-technical-report/04-streaming.md`<br>`sources/docs/langgraph-technical-report/05-human-in-the-loop.md` |
| [LangGraph 多智能体系统](../wiki/concepts/langgraph-multi-agent.md) | `sources/docs/langgraph-technical-report/06-multi-agent.md` |
| [LangGraph 实现方法](../wiki/summaries/langgraph-implementation.md) | `sources/docs/langgraph-technical-report/07-technical-plan-and-examples.md` |
| [LangGraph 定位与边界](../wiki/summaries/langgraph-positioning.md) | `sources/docs/langgraph-technical-report/08-summary-and-comparison.md` |

## OpenClaw 架构与记忆

| 聚合专题 | 来源集合 |
|---|---|
| [OpenClaw 架构总览](../wiki/summaries/openclaw-architecture-overview.md) | `sources/openclaw/architecture/core-architecture.md`<br>`sources/openclaw/architecture/openclaw-architecture-and-clone-guide.md` |
| [OpenClaw 多智能体运行时](../wiki/concepts/openclaw-multi-agent-runtime.md) | `sources/openclaw/architecture/multi-agent.md`<br>`sources/openclaw/best-practices/sessions-spawn-async-monitoring.md` |
| [OpenClaw 记忆架构](../wiki/summaries/openclaw-memory-architecture.md) | `sources/openclaw/AI 代理精英记忆架构最佳实践.md`<br>`sources/openclaw/architecture/memory-search-implementation.md` |
| [OpenClaw 记忆插件选型](../wiki/decisions/openclaw-memory-plugin-selection.md) | `sources/openclaw/memory-lancedb-comparison.md`<br>`sources/openclaw/memory-plugins-complete-guide.md`<br>`sources/openclaw/memory-search-best-practices.md`<br>`sources/openclaw/best-practices/openclaw-memory-plugins-guide.md` |
| [OpenClaw 记忆搜索](../wiki/summaries/openclaw-memory-search.md) | `sources/openclaw-memory-search-guide.md` |
| [OpenClaw 记忆搜索操作](../wiki/summaries/openclaw-memory-search-operations.md) | `sources/openclaw/memory-search-quickstart.md`<br>`sources/openclaw/openclaw-memory-index-repair.md` |

## OpenClaw 配置运维

| 聚合专题 | 来源集合 |
|---|---|
| [OpenClaw 运行时配置](../wiki/summaries/openclaw-runtime-configuration.md) | `sources/openclaw/best-practices/openclaw-configuration-best-practices.md`<br>`sources/openclaw/environment-variables-guide.md`<br>`sources/openclaw/sandbox-configuration-practice.md` |
| [OpenTask 容器运维](../wiki/summaries/openclaw-opentask-container-operations.md) | `sources/openclaw/OpenTask-容器配置指南.md` |
| [OpenClaw 安装与运行时管理](../wiki/summaries/openclaw-installation-and-runtime-management.md) | `sources/openclaw/best-practices/docker-quickstart.md`<br>`sources/openclaw/best-practices/nvm-nodejs-openclaw-management.md` |
| [百炼模型配置](../wiki/summaries/openclaw-bailian-model-configuration.md) | `sources/openclaw/claude-code-bailian-coding-plan.md`<br>`sources/openclaw/best-practices/bailian-coding-plan-complete-guide.md`<br>`sources/openclaw/best-practices/bailian-model-configuration.md`<br>`sources/openclaw/best-practices/bailian-models-best-practices.md` |
| [网络与远程访问](../wiki/summaries/openclaw-network-and-remote-access.md) | `sources/openclaw/solutions/macos-tcp-connection-storm.md`<br>`sources/openclaw/solutions/control-ui-http-remote-access.md` |

## 平台集成

| 聚合专题 | 来源集合 |
|---|---|
| [Claude Code 与 Telegram 远程通信](../wiki/summaries/claude-code-telegram-remote.md) | `sources/claude-code-telegram-remote-communication.md` |
| [Playwright 自动化](../wiki/summaries/openclaw-playwright-automation.md) | `sources/openclaw/Playwright-浏览器自动化方案.md`<br>`sources/openclaw/best-practices/playwright-best-practices.md` |
| [Discord 运维](../wiki/summaries/openclaw-discord-operations.md) | `sources/openclaw/discord-bot-best-practices.md`<br>`sources/openclaw/best-practices/discord-bot-configuration-best-practices.md`<br>`sources/openclaw/best-practices/discord-multi-bot-management.md` |
| [群聊与 Telegram 路由](../wiki/summaries/openclaw-group-chat-and-telegram-routing.md) | `sources/openclaw/best-practices/group-chat-configuration-best-practices.md`<br>`sources/openclaw/best-practices/multi-agent-telegram.md`<br>`sources/openclaw/solutions/telegram-multi-agent-routing.md` |
| [Telegram 连通性排障](../wiki/summaries/openclaw-telegram-connectivity-troubleshooting.md) | `sources/openclaw/solutions/cc-connect-telegram-unresponsive.md`<br>`sources/openclaw/solutions/docker-telegram-proxy-connection-issue.md`<br>`sources/openclaw/solutions/proxy-solutions.md` |
| [GitHub 工作流](../wiki/summaries/openclaw-github-workflow.md) | `sources/openclaw/best-practices/github-workflow-best-practices.md` |

## 其他项目

| 聚合专题 | 来源集合 |
|---|---|
| [Hermes Agent](../wiki/entities/hermes-agent.md) | `sources/hermes-agent/overview.md` |
| [OpenHorse 配置](../wiki/summaries/openhorse-configuration.md) | `sources/openhorse/best-practices/openhorse-configuration-best-practices.md` |
| [Fork 维护实践](../wiki/summaries/fork-maintenance.md) | `sources/docs/fork-maintenance-best-practices.md` |
| 历史项目入口 | `sources/README.md`<br>`sources/index.md` |
