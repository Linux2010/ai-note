# AI-Note 索引

## 项目结构

```
ai-note/
├── README.md                          # 项目介绍
├── index.md                           # 本索引文件
├── claude-code-telegram-remote-communication.md  # Telegram 远程通信
├── openclaw-memory-search-guide.md    # OpenClaw 记忆搜索指南
├── docs/                              # 综合技术文档
│   ├── elite-longterm-memory-guide.md
│   ├── fork-maintenance-best-practices.md
│   └── langgraph-technical-report/
├── hermes-agent/                      # Hermes Agent 文档
│   └── overview.md
└── openclaw/                          # OpenClaw 框架文档
    ├── [根目录文档]
    ├── architecture/                  # 架构设计
    ├── best-practices/                # 最佳实践
    └── solutions/                     # 技术解决方案
```

---

## 文档分类

### OpenClaw

#### 架构设计
- [OpenClaw 架构设计与复刻指南](openclaw/architecture/openclaw-architecture-and-clone-guide.md)
- [多代理架构完整方案](openclaw/architecture/multi-agent.md)
- [核心架构与消息处理循环](openclaw/architecture/core-architecture.md)
- [记忆搜索实现方案](openclaw/architecture/memory-search-implementation.md)

#### 技术解决方案
- [macOS TCP 连接风暴排查](openclaw/solutions/macos-tcp-connection-storm.md)
- [代理问题解决方案](openclaw/solutions/proxy-solutions.md)
- [Telegram 多频道路由配置](openclaw/solutions/telegram-multi-agent-routing.md)
- [Control UI HTTP 远程访问配置](openclaw/solutions/control-ui-http-remote-access.md)
- [Docker Telegram 代理连接问题](openclaw/solutions/docker-telegram-proxy-connection-issue.md)
- [cc-connect Telegram Bot 无响应诊断与修复](openclaw/solutions/cc-connect-telegram-unresponsive.md)
- [Memory Search 索引问题修复](openclaw/openclaw-memory-index-repair.md)

#### 最佳实践
- [异步任务监控方案](openclaw/best-practices/sessions-spawn-async-monitoring.md)
- [多代理 Telegram 交互](openclaw/best-practices/multi-agent-telegram.md)
- [OpenClaw 配置最佳实践](openclaw/best-practices/openclaw-configuration-best-practices.md)
- [Memory 插件对比指南](openclaw/best-practices/openclaw-memory-plugins-guide.md)
- [Discord Bot 配置最佳实践](openclaw/best-practices/discord-bot-configuration-best-practices.md)
- [Discord 多 Bot 管理 - 主 Agent 方案](openclaw/best-practices/discord-multi-bot-management.md)
- [OpenClaw Docker 快速部署指南](openclaw/best-practices/docker-quickstart.md)
- [百炼 Qwen 模型配置指南](openclaw/best-practices/bailian-model-configuration.md)
- [百炼平台大模型配置最佳实践](openclaw/best-practices/bailian-models-best-practices.md)
- [百炼 Coding Plan 完整指南](openclaw/best-practices/bailian-coding-plan-complete-guide.md)
- [NVM + Node.js + OpenClaw 版本管理](openclaw/best-practices/nvm-nodejs-openclaw-management.md)
- [Playwright 浏览器自动化最佳实践](openclaw/best-practices/playwright-best-practices.md)
- [GitHub Workflow 最佳实践](openclaw/best-practices/github-workflow-best-practices.md)
- [群组对话配置最佳实践](openclaw/best-practices/group-chat-configuration-best-practices.md)

#### OpenClaw 根目录文档
- [AI 代理精英记忆架构最佳实践](openclaw/AI 代理精英记忆架构最佳实践.md)
- [OpenTask 容器配置指南](openclaw/OpenTask-容器配置指南.md)
- [Playwright 浏览器自动化方案](openclaw/Playwright-浏览器自动化方案.md)
- [Claude Code 百炼 Coding Plan](openclaw/claude-code-bailian-coding-plan.md)
- [Discord Bot 最佳实践](openclaw/discord-bot-best-practices.md)
- [环境变量配置指南](openclaw/environment-variables-guide.md)
- [Memory LanceDB 对比分析](openclaw/memory-lancedb-comparison.md)
- [Memory 插件完整指南](openclaw/memory-plugins-complete-guide.md)
- [记忆搜索最佳实践](openclaw/memory-search-best-practices.md)
- [记忆搜索快速入门](openclaw/memory-search-quickstart.md)
- [沙箱配置实践](openclaw/sandbox-configuration-practice.md)

---

### Hermes Agent
- [核心特点评估](hermes-agent/overview.md)

---

### 综合技术文档 (docs/)

#### 长期记忆系统
- [Elite Longterm Memory 完整实现指南](docs/elite-longterm-memory-guide.md)

#### LangGraph 技术报告
- [概述与架构](docs/langgraph-technical-report/01-overview-and-architecture.md)
- [核心概念](docs/langgraph-technical-report/02-core-concepts.md)
- [持久化与检查点](docs/langgraph-technical-report/03-persistence-and-checkpoints.md)
- [流式输出](docs/langgraph-technical-report/04-streaming.md)
- [人在回路](docs/langgraph-technical-report/05-human-in-the-loop.md)
- [多代理](docs/langgraph-technical-report/06-multi-agent.md)
- [技术计划与示例](docs/langgraph-technical-report/07-technical-plan-and-examples.md)
- [总结与对比](docs/langgraph-technical-report/08-summary-and-comparison.md)
- [README](docs/langgraph-technical-report/README.md)

#### Agent-CLI 语言选型
- [Agent-CLI 最适合的开发语言：全面对比与选型指南](docs/agent-cli-language-comparison.md)

#### 其他
- [Fork 维护最佳实践](docs/fork-maintenance-best-practices.md)

---

### 根目录文档
- [Telegram 远程通信指南](claude-code-telegram-remote-communication.md)
- [OpenClaw 记忆搜索指南](openclaw-memory-search-guide.md)

---

## 文档规范

所有文档遵循以下元数据结构：
```yaml
文档类型: [架构设计|技术解决方案|最佳实践]
适用场景: [具体使用场景]
最后更新: YYYY-MM-DD
```

## 贡献流程

1. 在对应领域目录下创建文档
2. 更新本索引文件
3. 提交 PR 到主仓库
4. 确保文档 AI 友好（结构化、可解析）

---
**索引最后更新**: 2026-07-19
