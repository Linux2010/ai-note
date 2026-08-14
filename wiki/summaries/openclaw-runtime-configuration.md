---
id: summary-openclaw-runtime-configuration
type: source-summary
title: "OpenClaw 运行时配置体系"
status: draft
updated: 2026-08-14
owner: "llm-wiki-expert"
sources:
  - ../../raw/sources/openclaw/best-practices/openclaw-configuration-best-practices.md
  - ../../raw/sources/openclaw/environment-variables-guide.md
  - ../../raw/sources/openclaw/sandbox-configuration-practice.md
---

# OpenClaw 运行时配置体系

## 概览

OpenClaw 的运行时配置不是单一文件，而是由全局 Gateway 配置、Agent 级覆盖、工作区技能和进程环境共同组成。重构配置时应先区分“系统级路由与通道”“Agent 权限与沙箱”“技能装载”和“密钥注入”四类职责，再决定配置落点，避免把字段写入不支持的位置或让不同来源相互覆盖。

本页综合的原文面向 OpenClaw 2026.3 至 2026.4.x，字段名称和加载行为具有版本边界。升级后应以当前配置参考和实际运行结果为准。

## 核心内容与机制

### 配置分层

- `~/.openclaw/openclaw.json` 承担 Gateway、Channel、模型 Provider、Agent 列表、绑定规则和全局技能设置。
- `~/.openclaw/agents/<agentId>/config.json` 用于 Agent 级沙箱和工具权限覆盖；原文明确指出 `skills` 不是该文件的有效字段。
- `<workspace>/skills/` 提供 Agent 专属技能，`~/.openclaw/skills/` 提供全局共享技能，内置技能优先级最低。
- `~/.openclaw/.env`、进程环境和 `openclaw.json` 的 `env` 块共同提供运行时变量与密钥。

### 环境变量优先级

原文给出的加载顺序是：父进程环境、当前工作目录 `.env`、全局 `~/.openclaw/.env`、配置文件 `env` 块、可选的 login-shell 导入。其共同原则是后续来源只补充缺失值，不覆盖已存在的变量。因此，同名变量异常时，应先检查真正启动 Gateway 的进程环境，而不是只检查磁盘上的 `.env`。

### 权限与沙箱

沙箱和工具权限是两个相关但不同的控制面。关闭沙箱会扩大文件系统和进程访问范围，而 `tools.allow/deny` 决定 Agent 能调用哪些能力。原文包含为可信开发 Agent 关闭沙箱的实践，但这不是通用默认值；面向公共入口、财务或其他敏感任务的 Agent 应保留隔离并遵循最小权限。

## 实践要点与决策依据

- 路由、Channel、模型和 Agent 身份集中放入 `openclaw.json`，避免在多个 Agent 文件中复制系统级配置。
- Agent 差异优先通过 `config.json` 的沙箱和工具字段、工作区技能目录以及 `agents.list[]` 表达。
- 密钥优先放入权限为 `600` 的全局 `.env` 或 SecretRef，不在示例、仓库和 Agent 提示词中硬编码。
- 多容器部署要明确 `.env` 是通过卷挂载还是容器环境注入；宿主机文件存在并不代表容器可见。
- 修改后是否需要重启取决于配置项和当前版本的 watcher 能力，不能假设所有字段都会热加载。
- 关闭沙箱属于风险接受决策，应记录适用 Agent、所需外部路径、允许工具和恢复方式，而不是把 `mode: off` 当成排障捷径。

## 相关主题

- [[summaries/openclaw-installation-and-runtime-management|OpenClaw 安装与运行时管理]]
- [[summaries/openclaw-bailian-model-configuration|OpenClaw 百炼模型配置]]
- [[summaries/openclaw-network-and-remote-access|OpenClaw 网络与远程访问]]
- [[summaries/openclaw-github-workflow|OpenClaw GitHub 工作流]]
