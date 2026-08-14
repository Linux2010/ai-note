---
id: decision-openclaw-memory-plugin-selection
type: decision
title: "OpenClaw 记忆插件选型"
status: draft
updated: 2026-08-14
owner: "llm-wiki-expert"
sources:
  - ../../raw/sources/openclaw/memory-lancedb-comparison.md
  - ../../raw/sources/openclaw/memory-plugins-complete-guide.md
  - ../../raw/sources/openclaw/memory-search-best-practices.md
  - ../../raw/sources/openclaw/best-practices/openclaw-memory-plugins-guide.md
---

# OpenClaw 记忆插件选型

## 概览

记忆插件选型应围绕可读性、自动捕获、检索质量、外部资料接入、用户建模和运维成本进行，而不是只比较单一性能指标。四份来源覆盖 `memory-core`、LanceDB 系列、QMD 和 Honcho 等方案，但对推荐优先级存在差异，因此本页采用条件式决策，不声明一个插件适合所有场景。[LanceDB 对比](../../openclaw/memory-lancedb-comparison.md) [插件完全指南](../../openclaw/memory-plugins-complete-guide.md) [Memory Search 实践](../../openclaw/memory-search-best-practices.md) [插件对比指南](../../openclaw/best-practices/openclaw-memory-plugins-guide.md)

## 决策

- **默认基线**：当人类可读、可编辑、可用版本控制审查的 Markdown 记忆最重要时，优先采用 `memory-core` 一类文件中心方案。
- **自动对话记忆**：需要自动捕获、自动召回、重排和遗忘治理，并能承担额外依赖与运维成本时，评估 LanceDB 增强方案。
- **外部资料检索**：主要需求是索引外部文件或会话转录时，单独评估 QMD 路线。
- **跨会话用户建模**：需要持续用户模型而不仅是文档召回时，再评估 Honcho 路线。

该决策必须以当前安装版本、插件可用性和实际检索评测为前置条件，不能直接沿用来源中的历史性能表或推荐结论。

## 核心机制与取舍

### 文件中心方案

文件中心方案以 Markdown 为长期事实载体，并从文件构建全文或向量索引。优势是可读、可审查、容易迁移和备份；不足是自动捕获与高级生命周期治理较弱。[插件完全指南](../../openclaw/memory-plugins-complete-guide.md) [Memory Search 实践](../../openclaw/memory-search-best-practices.md)

### LanceDB 增强方案

LanceDB 系列把对话记忆写入专用存储，并提供自动捕获、召回和删除等能力；增强版本还强调混合检索、重排与智能治理。代价是依赖、配置和故障面增加，插件存储中的内容也不天然具备 Markdown 的可读性。[LanceDB 对比](../../openclaw/memory-lancedb-comparison.md) [插件完全指南](../../openclaw/memory-plugins-complete-guide.md)

### 专用检索与用户模型

QMD 更偏向外部资料和转录检索，Honcho 更偏向跨会话用户建模。它们解决的问题与基础文件记忆不完全相同，不应仅凭“检索更强”替换现有记忆主存储。[Memory Search 实践](../../openclaw/memory-search-best-practices.md) [插件完全指南](../../openclaw/memory-plugins-complete-guide.md)

## 决策依据与实践要点

- 先定义主要任务：知识库维护、对话记忆、外部文档检索或用户建模。
- 建立固定问题集，比较召回正确性、来源可追溯性、延迟和维护工作量。
- 明确记忆槽位和主存储；不要假设多个插件可以无冲突地同时担任主记忆实现。
- 迁移前保留 `MEMORY.md` 与 `memory/*.md`，并把插件数据库视为需单独备份和验证的资产。
- 核对 embedding provider、模型、维度和兼容端点，配置不一致可能导致索引不可用。[插件对比指南](../../openclaw/best-practices/openclaw-memory-plugins-guide.md)

## 证据与来源

- [Memory LanceDB 插件深度对比分析](../../openclaw/memory-lancedb-comparison.md)：支持文件记忆与 LanceDB 系列的能力、可读性和运维取舍。
- [OpenClaw Memory 插件完全指南](../../openclaw/memory-plugins-complete-guide.md)：支持多方案分类、部署场景和迁移考虑。
- [OpenClaw Memory-Search 插件完全指南](../../openclaw/memory-search-best-practices.md)：支持检索方案、使用场景和故障因素比较。
- [OpenClaw Memory 插件对比指南](../../openclaw/best-practices/openclaw-memory-plugins-guide.md)：支持基础插件配置、槽位差异和迁移注意事项。

## 冲突与限制

来源对候选数量、插件命名和默认推荐并不完全一致，也包含可能随版本变化的性能与成本信息。当前决策只提炼稳定的选择维度；具体插件是否可用、是否兼容当前配置以及性能是否成立，仍需在迁移时重新验证。

## 相关主题

- [OpenClaw 记忆架构](../summaries/openclaw-memory-architecture.md)
- [OpenClaw Memory Search 运维](../summaries/openclaw-memory-search-operations.md)
- [OpenClaw 多代理运行时](../concepts/openclaw-multi-agent-runtime.md)
