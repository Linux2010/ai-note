# OpenClaw 异步任务监控方案科普

> **文档类型**: 技术解决方案  
> **适用场景**: 长时间任务、并行任务、任务编排  
> **最后更新**: 2026-03-13  
> **作者**: Main Agent (OpenClaw)  
> **标签**: sessions_spawn, subagent, async, monitoring

---

## 📋 目录

1. [需求背景](#需求背景)
2. [官方机制](#官方机制)
3. [完整方案](#完整方案)
4. [参数配置](#参数配置)
5. [监控方式](#监控方式)
6. [最佳实践](#最佳实践)
7. [示例代码](#示例代码)

---

## 🎯 需求背景

### 用户需求

用户希望实现：
1. ✅ 提交一个**异步任务**
2. ✅ **定时监控**任务状态
3. ✅ 完成后**汇报给主会话**

### 常见场景

| 场景 | 说明 |
|------|------|
| 长时间研究 | 需要数分钟的研究分析任务 |
| 并行处理 | 多个独立任务同时执行 |
| 后台任务 | 不阻塞主会话的其他工作 |
| 任务编排 | 复杂的多步骤工作流 |

---

## 📖 官方机制

### 核心设计哲学

> "Sub-agents run in their own session, and when finished, **announce their result back to the requester chat channel**."

OpenClaw 采用**推送式通知**而非**轮询式监控**：

```
❌ 不推荐：主会话定时 polling 子代理状态
✅ 推荐：子代理完成后自动 announce 回主会话
```

### 官方文档来源

- Session Tools: https://docs.openclaw.ai/concepts/session-tool
- Sub-Agents: https://docs.openclaw.ai/tools/subagents
- Configuration: https://docs.openclaw.ai/gateway/configuration-reference

---

## 🔄 完整方案

### 工作流程

```
┌─────────────────┐
│   Main Session  │
│    (主会话)      │
└────────┬────────┘
         │ 1. sessions_spawn({ task: "..." })
         │    立即返回 { status: "accepted", runId, childSessionKey }
         ▼
┌─────────────────────────────────┐
│     Sub-agent Session           │
│    (隔离会话，独立上下文)        │
│                                 │
│   执行任务... (可能几分钟)       │
│   使用独立工具集                 │
│   独立 token 计费                │
└────────┬────────────────────────┘
         │ 2. 任务完成
         │ 3. 自动执行 Announce Step
         ▼
┌─────────────────┐
│   Main Session  │
│    (主会话)      │
│                 │
│ 📬 收到完成通知：│
│ - Status 状态   │
│ - Result 结果   │
│ - Stats 统计    │
└─────────────────┘
```

### 关键特性

| 特性 | 说明 |
|------|------|
| **非阻塞** | `sessions_spawn` 立即返回，不等待任务完成 |
| **自动通知** | 完成后自动 announce 回主会话 |
| **隔离执行** | 子代理在独立会话中运行，互不干扰 |
| **状态追踪** | 可通过 `runId` 和 `childSessionKey` 查询 |

---

## ⚙️ 参数配置

### 完整参数定义

```typescript
sessions_spawn({
  // ========== 必填 ==========
  task: string,                    // 任务描述（必填）
  
  // ========== 可选配置 ==========
  label?: string,                  // 任务标签（用于日志/UI）
  agentId?: string,                // 指定目标 agent ID
  model?: string,                  // 覆盖子代理模型
  thinking?: string,               // 思考级别
  runTimeoutSeconds?: number,      // 超时时间（秒）
  thread?: boolean,                // 线程绑定（Discord 支持）
  mode?: "run" | "session",        // 执行模式
  cleanup?: "delete" | "keep",     // 清理策略
  sandbox?: "inherit" | "require", // 沙箱要求
  attachments?: Array<{            // 文件附件
    name: string,
    content: string,
    encoding?: "utf8" | "base64",
    mimeType?: string
  }>
})
```

### 关键参数说明

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `mode` | `"run"` | `"run"`=一次性，`"session"`=持久会话 |
| `runTimeoutSeconds` | `0` (无限制) | 超时后自动中止子代理 |
| `cleanup` | `"keep"` | `"delete"`=完成后归档 |
| `model` | 继承主会话 | 可为子代理指定更便宜的模型 |

---

## 🔍 监控方式

### 方式 1：自动通知（推荐）

**无需任何操作**，子代理完成后自动发送通知到主会话。

通知内容包含：
```
┌─────────────────────────────────────────┐
│  📬 Sub-agent 完成通知                    │
├─────────────────────────────────────────┤
│  Status: completed successfully         │
│  Result: [子代理的最终回复]              │
│                                         │
│  📊 统计信息：                           │
│  - Runtime: 5m12s                       │
│  - Tokens: 12,450 / 3,200               │
│  - Session: agent:main:subagent:uuid    │
└─────────────────────────────────────────┘
```

### 方式 2：手动检查状态

```javascript
// 列出所有子代理
subagents(action="list")

// 返回示例：
// - agent-abc123: "数据分析任务" (running, 2min)
// - agent-def456: "PR 分析" (completed)
```

### 方式 3：查看子代理日志

```javascript
// 查看特定子代理的运行日志
subagents(action="log", target="agent-abc123", limit: 50)

// 或查看会话历史
sessions_history({
  sessionKey: "agent:main:subagent:uuid",
  limit: 100,
  includeTools: true
})
```

### 方式 4：Slash 命令

```bash
# 列出所有子代理
/subagents list

# 查看特定子代理详情
/subagents info <id>

# 查看日志
/subagents log <id> [limit] [tools]
```

---

## 💡 最佳实践

### 方案对比

| 方案 | 优点 | 缺点 | 推荐度 |
|------|------|------|--------|
| **自动通知** | 简单、高效、无需轮询 | Gateway 重启可能丢失通知 | ⭐⭐⭐⭐⭐ |
| **手动轮询** | 可控性强 | 浪费 token、复杂 | ⭐⭐ |
| **混合模式** | 自动通知 + 超时检查 | 适中复杂度 | ⭐⭐⭐⭐ |

### 推荐实现

```javascript
// 1. 提交任务
const { runId, childSessionKey } = await sessions_spawn({
  task: "分析这个数据集，生成详细报告",
  runTimeoutSeconds: 600,  // 10 分钟超时
  label: "数据分析任务"
});

// 2. 记录 runId 用于后续追踪
console.log(`任务已提交：${runId}, ${childSessionKey}`);

// 3. 等待自动通知（无需做任何事！）
// 子代理完成后会自动发送完成消息到主会话
```

### 注意事项

1. **自动归档**: 子代理会话默认 60 分钟后自动归档
2. **Best-Effort 通知**: Gateway 重启可能导致待发送通知丢失
3. **工具限制**: 子代理默认不能使用 session 工具
4. **成本考虑**: 每个子代理有独立的 token 使用

---

## 📝 示例代码

### 示例 1：基础异步任务

```javascript
// 提交异步任务
sessions_spawn({
  task: "研究 OpenClaw 最新的 memory 技能，整理成表格",
  label: "技能研究",
  runTimeoutSeconds: 300
});

// 立即返回，继续处理其他事情
// 5 分钟后自动收到完成通知
```

### 示例 2：带超时的长时间任务

```javascript
// 长时间运行的任务
sessions_spawn({
  task: "分析过去一年的销售数据，生成趋势报告和可视化",
  label: "年度销售分析",
  runTimeoutSeconds: 1800,  // 30 分钟超时
  model: "qwen3-max-thinking",
  cleanup: "keep"  // 保留会话用于后续查看
});
```

### 示例 3：指定 Agent 执行

```javascript
// 让 Stock Agent 分析持仓
sessions_spawn({
  agentId: "stock",
  task: "分析当前持仓风险，给出调仓建议",
  model: "qwen3-max-thinking",
  thinking: "high"
});

// 让 Coding Agent 处理 GitHub PR
sessions_spawn({
  agentId: "coding",
  task: "检查最近的 PR 评论，回复需要回应的",
  mode: "run"
});
```

### 示例 4：Orchestrator 模式（嵌套子代理）

```javascript
// 配置允许两层嵌套
{
  "agents": {
    "defaults": {
      "subagents": {
        "maxSpawnDepth": 2,
        "maxChildrenPerAgent": 5,
        "maxConcurrent": 8
      }
    }
  }
}

// 主会话 → Orchestrator
sessions_spawn({
  task: "协调多个子任务：1) 数据收集 2) 数据分析 3) 生成报告",
  label: "数据管道协调",
  mode: "run"
});

// Orchestrator 内部再 spawn 多个 worker
```

---

## 🔧 相关命令速查

| 命令 | 说明 |
|------|------|
| `/subagents list` | 列出所有子代理 |
| `/subagents info <id>` | 查看子代理详情 |
| `/subagents log <id>` | 查看子代理日志 |
| `/subagents kill <id>` | 终止子代理 |
| `/subagents send <id> <msg>` | 发送消息给子代理 |
| `/subagents spawn <agent> <task>` | 手动 spawn 子代理 |
| `/stop` | 停止当前会话和所有子代理 |

---

## 📊 架构细节

### 会话 Key 格式

```
Depth 0 (Main):  agent:<agentId>:main
Depth 1 (Sub):   agent:<agentId>:subagent:<uuid>
Depth 2 (Worker): agent:<agentId>:subagent:<uuid>:subagent:<uuid>
```

### 工具权限矩阵

| 工具 | Main | Depth 1 (Orchestrator) | Depth 2 (Worker) |
|------|------|------------------------|------------------|
| `sessions_spawn` | ✅ | ✅ (maxSpawnDepth≥2) | ❌ |
| `sessions_list` | ✅ | ✅ (maxSpawnDepth≥2) | ❌ |
| `sessions_history` | ✅ | ✅ (maxSpawnDepth≥2) | ❌ |
| `sessions_send` | ✅ | ❌ | ❌ |
| 其他工具 | ✅ | ✅ | ✅ |

### 并发控制

```json5
{
  "agents": {
    "defaults": {
      "subagents": {
        "maxConcurrent": 8,        // 全局并发限制
        "maxChildrenPerAgent": 5,  // 每个 agent 最多 5 个子代理
        "runTimeoutSeconds": 900,  // 默认超时 15 分钟
        "archiveAfterMinutes": 60  // 自动归档时间
      }
    }
  }
}
```

---

## ⚠️ 限制与注意事项

### 已知限制

1. **通知丢失风险**: Gateway 重启时，待发送的 announce 可能丢失
2. **嵌套深度**: 最大支持 5 层嵌套（推荐 2 层）
3. **工具限制**: 子代理默认不能使用 session 工具
4. **资源共享**: 所有子代理共享同一 Gateway 进程资源

### 安全考虑

1. **沙箱隔离**: 使用 `sandbox: "require"` 确保子代理在沙箱中运行
2. **Agent 允许列表**: 配置 `agents.list[].subagents.allowAgents` 限制可 target 的 agents
3. **禁止嵌套 spawn**: 默认 `maxSpawnDepth: 1`，防止无限嵌套

---

## 📖 总结

### ✅ 官方支持的方式

1. 使用 `sessions_spawn` 提交异步任务
2. 无需手动轮询，完成后自动通知
3. 可通过 `subagents list` 手动检查状态（如需要）
4. 支持超时、嵌套、并发控制

### ❌ 不推荐的方式

1. 手动轮询检查状态（浪费 token）
2. 在主会话中等待子代理完成（阻塞）
3. 子代理内再 spawn 子代理（默认禁止）

### 🎯 最佳实践

```
1. sessions_spawn 提交任务 → 立即返回
2. 继续处理其他事情
3. 等待自动通知（推送式）
4. 如需要，用 subagents list 检查进度
```

---

## 🔗 相关链接

- **官方文档**: https://docs.openclaw.ai/tools/subagents
- **Session Tools**: https://docs.openclaw.ai/concepts/session-tool
- **配置参考**: https://docs.openclaw.ai/gateway/configuration-reference
- **GitHub 仓库**: https://github.com/openclaw/openclaw

---

*本文档遵循 AI-Note 规范，旨在提供清晰、可执行的 OpenClaw 异步任务监控方案。*
