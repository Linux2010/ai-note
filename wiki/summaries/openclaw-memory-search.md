---
id: summary-openclaw-memory-search
type: summary
title: "OpenClaw Memory Search"
status: draft
updated: 2026-08-14
owner: "llm-wiki-expert"
sources:
  - ../../raw/sources/openclaw-memory-search-guide.md
---

# OpenClaw Memory Search

## 概览

OpenClaw Memory Search 为 `MEMORY.md` 和 `memory/*.md` 等文件建立向量索引，在对话前按语义召回相关片段。它解决的是文件知识检索，不等同于自动保存全部对话历史；对话自动捕获由独立的 memory-lancedb 插件承担，两套机制可以并存。

## 核心机制

Memory Search 从配置的 `sources` 和 `extraPaths` 读取 Markdown 内容，通过指定 Embedding Provider 生成向量并维护索引。查询时将用户问题向量化，返回语义相近的记忆片段。`sync.watch` 可以在源文件变化时触发重新索引，Fallback 则在主 Provider 不可用时提供降级路径。

| 机制 | 索引对象 | 主要用途 |
|---|---|---|
| memory-core / Memory Search | `MEMORY.md`、日期记忆和扩展路径 | 检索经过整理的文件知识 |
| memory-lancedb | 对话历史 | 自动捕获和召回动态会话内容 |

## Provider 决策

托管 Provider 配置简单、质量稳定，但需要处理凭据、成本和数据边界；Ollama 等本地方案更适合隐私敏感或离线环境，但需要本地资源和模型运维。Docker 中访问宿主机服务时不能默认使用容器自身的 `localhost`，应配置可解析的宿主机地址。

## 实践要点

- 保持 `MEMORY.md` 精炼，把日期性和低频信息归档到分文件。
- 根据语言、质量、维度和成本选择 Embedding 模型，不只看是否能返回向量。
- 开启文件监控后仍需观察索引失败和延迟，不能假设修改一定已生效。
- 为主 Provider 配置可实际工作的 Fallback，并验证向量维度变化时的重建策略。
- 扩展路径只加入可信 Markdown，避免把敏感文件或巨大生成目录纳入索引。
- 定期用已知问题测试召回质量，并检查结果是否能够追溯到原文件。

## 排错顺序

搜索无结果时，依次确认功能开关、源文件存在性、Embedding 端点、模型名称和索引状态。容器连接本地 Ollama 失败时先检查网络地址。索引过大时，应先清理过时记忆和不必要路径，再考虑降低向量维度。

## 限制与边界

语义相近不等于事实正确。召回结果仍需要来源路径、时间和上下文来判断适用性；自动索引也不能替代记忆冲突处理、删除策略和隐私治理。

## 相关主题

- [[concepts/agent-long-term-memory|Agent 长期记忆分层架构]]
- [[summaries/openhorse-configuration|OpenHorse 配置与最佳实践]]
- [[entities/hermes-agent|Hermes Agent]]
