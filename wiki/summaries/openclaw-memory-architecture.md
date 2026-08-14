---
id: summary-openclaw-memory-architecture
type: summary
title: "OpenClaw 记忆架构"
status: draft
updated: 2026-08-14
owner: "llm-wiki-expert"
sources:
  - ../../raw/sources/openclaw/AI 代理精英记忆架构最佳实践.md
  - ../../raw/sources/openclaw/architecture/memory-search-implementation.md
---

# OpenClaw 记忆架构

## 概览

这组来源从两个层面描述代理记忆：实践文档提出按生命周期组织记忆的分层模型，技术文档解释 OpenClaw Memory Search 如何索引 Markdown 记忆并执行混合检索。前者是知识治理方法，后者是检索实现，两者共同形成“可读源文件、持久索引、按需召回”的工作闭环。[分层记忆实践](../../openclaw/AI%20代理精英记忆架构最佳实践.md) [检索实现](../../openclaw/architecture/memory-search-implementation.md)

## 核心内容与机制

### 分层记忆

实践来源把记忆划分为活跃会话状态、语义检索层、长期结构化知识、人类整理档案和备份层。其核心不是要求所有部署采用同一存储产品，而是让当前任务、稳定事实、历史记录与灾难恢复拥有不同更新节奏。[分层记忆实践](../../openclaw/AI%20代理精英记忆架构最佳实践.md)

### 可追溯记忆源

`MEMORY.md` 和 `memory/*.md` 等文件承担人类可读的记忆来源。写入时应区分会话状态、长期偏好、决策和经验，避免把全部上下文堆入单一文件。重要信息先持久化再继续任务，可以降低会话中断造成的信息丢失。[分层记忆实践](../../openclaw/AI%20代理精英记忆架构最佳实践.md)

### 索引与混合检索

Memory Search 将来源文件切分并写入索引，通过向量相似度和全文关键词两条路径检索，再进行分数融合与去重，最后返回带原始文件位置的片段。语义检索提升近义表达召回，关键词检索保留名称、命令和精确术语的匹配能力。[检索实现](../../openclaw/architecture/memory-search-implementation.md)

### Agent 隔离与配置继承

检索实现按 Agent 维护记忆索引，并允许默认配置与 Agent 级覆盖协作。这使共享运行时可以采用统一策略，同时避免不同代理的记忆数据无条件混合。[检索实现](../../openclaw/architecture/memory-search-implementation.md)

## 实践要点

- 会话开始时恢复活跃状态，任务过程中及时记录关键决策，不把持久化推迟到会话结束。
- 长期记忆只保留跨会话仍有价值的信息，临时日志进入按日期或任务组织的记录。
- 同时保留可读 Markdown 与可重建索引；索引是派生数据，不应成为唯一事实来源。
- 调整 embedding、分块或检索参数后重新评估召回质量，并按需要重建索引。
- 多 Agent 部署应明确数据边界、默认配置和 Agent 级覆盖，避免意外共享记忆。

## 证据与来源

- [AI 代理精英记忆架构最佳实践](../../openclaw/AI%20代理精英记忆架构最佳实践.md)：支持分层记忆、写前记录、会话恢复和周期维护方法。
- [OpenClaw Memory Search 技术实现](../../openclaw/architecture/memory-search-implementation.md)：支持来源文件、索引结构、混合检索、结果去重和 Agent 隔离机制。

## 冲突与限制

分层实践使用 LanceDB 等组件说明语义记忆，技术实现则描述 SQLite、向量扩展与全文索引。它们可能对应不同插件或版本，不能合并为唯一固定存储栈。分层名称属于治理模型，也不证明每层都由 OpenClaw 原生自动维护。

## 相关主题

- [OpenClaw 记忆插件选型](../decisions/openclaw-memory-plugin-selection.md)
- [OpenClaw Memory Search 运维](openclaw-memory-search-operations.md)
- [OpenClaw 多代理运行时](../concepts/openclaw-multi-agent-runtime.md)
