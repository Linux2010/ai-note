---
id: summary-openclaw-installation-and-runtime-management
type: source-summary
title: "OpenClaw 安装与运行时管理"
status: draft
updated: 2026-08-14
owner: "llm-wiki-expert"
sources:
  - ../../raw/sources/openclaw/best-practices/docker-quickstart.md
  - ../../raw/sources/openclaw/best-practices/nvm-nodejs-openclaw-management.md
---

# OpenClaw 安装与运行时管理

## 概览

OpenClaw 常见的运行方式包括 nvm 管理的本机 Node.js 全局安装，以及使用 Docker 隔离的容器实例。前者适合单用户开发环境，后者适合多实例、端口隔离和可重复部署。选择方案时要同时考虑包版本、配置持久化、端口暴露、用户身份和升级方式。

## 核心内容与机制

### nvm 下的独立全局包

nvm 的每个 Node.js 版本拥有独立的全局 npm 包目录。切换 Node 版本时，命令可能“消失”，通常不是 OpenClaw 被卸载，而是当前 Node 环境没有安装该包。用户级状态目录可以共享，但可执行程序和包版本彼此独立。

原文推荐选择一个 LTS 版本作为 OpenClaw 主力运行时，仅维护一个全局安装。只有兼容性测试等明确需求，才值得在多个 Node 版本中重复安装。

### Docker 的持久化边界

原文镜像以 `node` 用户运行，容器内状态目录是 `/home/node/.openclaw`。把宿主机目录误挂载到 `/root/.openclaw` 会造成配置看似写入、重启后却丢失的假象。Gateway 默认内部端口需要映射到宿主机端口；若需从容器外访问，还要同步处理绑定地址、允许的 Origin 和认证。

### 实例生命周期

无论本机还是容器部署，都应把“程序版本”和“状态数据”分离：程序可重装或替换，状态目录必须备份并持续挂载。配置变更后重启对应 Gateway 或容器，再通过状态、日志、端口和模型列表逐层确认生效。

## 实践要点与决策依据

- 个人本机开发优先采用“单一 LTS + 单一 OpenClaw 全局安装”，降低升级和 PATH 排障成本。
- 多 Bot 或多职责实例优先容器化，并为每个实例分配独立状态目录、容器名和宿主机端口。
- 容器卷固定挂载到运行用户真实的状态目录；升级镜像前先备份宿主机状态。
- Gateway 对外绑定时必须同时配置认证和允许的 Origin，不能仅把监听地址改为 `0.0.0.0`。
- 不在初始化脚本中保留真实 API Key 或固定弱 Token；凭证应由环境或 Secret 管理注入。
- 容器继承 Docker Desktop 全局代理时，先确认代理确实可用；错误代理会让 Telegram、模型和安装操作同时失败。

## 相关主题

- [[summaries/openclaw-runtime-configuration|OpenClaw 运行时配置体系]]
- [[summaries/openclaw-bailian-model-configuration|OpenClaw 百炼模型配置]]
- [[summaries/openclaw-playwright-automation|OpenClaw Playwright 自动化]]
- [[summaries/openclaw-network-and-remote-access|OpenClaw 网络与远程访问]]
