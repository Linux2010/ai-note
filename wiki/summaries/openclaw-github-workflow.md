---
id: summary-openclaw-github-workflow
type: source-summary
title: "OpenClaw GitHub 项目工作流"
status: draft
updated: 2026-08-14
owner: "llm-wiki-expert"
sources:
  - ../../raw/sources/openclaw/best-practices/github-workflow-best-practices.md
---

# OpenClaw GitHub 项目工作流

## 概览

OpenClaw 处理 GitHub 项目的核心是把仓库读取、修改生成、人工审核和最终交付拆成可控阶段。来源文档基于一种特定沙箱模型：Agent 可以读取用户项目，但只能在自己的工作区写入，用户负责把结果应用到仓库并完成测试、提交和推送。该边界取决于当前 OpenClaw 权限配置，不能假设在所有实例中一致。

## 核心内容与机制

### 工作区隔离

用户项目目录作为分析来源，Agent 工作区作为修改产物区。Agent 先读取目标仓库、理解结构并形成计划，再在可写工作区生成完整文件或补丁。隔离可以阻止未审核修改直接进入真实仓库，但会增加同步和版本漂移风险。

### GitHub 信息面

`gh` CLI 可用于读取 Issue、PR 和仓库元数据，认证由本机凭证存储处理。凭证不应进入提示词、脚本输出或项目文件。任务中引用 Issue 或 PR 时，要记录仓库、编号和验收条件，避免只依据标题实施。

### 交付闭环

一个完整工作流包括：读取与分析、提出修改方案、在隔离位置生成变更、说明文件和行为差异、由用户审核并应用、在真实仓库测试，最后才提交和推送。测试通过、创建提交和远端交付是不同证据，不能互相替代。

## 实践要点与决策依据

- 开始前明确真实仓库路径、Agent 可写边界、目标分支和交付责任人。
- 大任务拆成可独立审核的小变更，保持源文件、目标文件和验收条件的映射。
- 隔离工作区中的产物应基于明确的源版本；仓库继续变化后，不能直接覆盖应用旧文件。
- 安全修复除代码变更外，还应提供威胁边界、回归风险和验证建议。
- GitHub Token 交给 `gh` 或系统凭证存储，不写入 `.env` 样例、仓库或 Agent 输出。
- 若当前实例已获真实仓库写权限，仍应保留最小改动、用户现有变更保护和显式提交边界，而不是跳过审核。

## 相关主题

- [[summaries/openclaw-runtime-configuration|OpenClaw 运行时配置体系]]
- [[summaries/openclaw-installation-and-runtime-management|OpenClaw 安装与运行时管理]]
- [[summaries/openclaw-network-and-remote-access|OpenClaw 网络与远程访问]]
