# 百炼平台 OpenClaw 大模型配置最佳实践

**版本**: 2026.4.2  
**创建时间**: 2026-04-06  
**平台**: 阿里云百炼 (DashScope)  
**标签**: `bailian` `dashscope` `models` `configuration` `best-practices`

---

## 📋 概述

本文档记录在 OpenClaw 中配置阿里云百炼平台大模型的最佳实践，包括模型选型、上下文长度、配置模板和使用建议。

---

## 🔧 完整配置模板

```json
{
  "models": {
    "providers": {
      "bailian": {
        "baseUrl": "https://coding.dashscope.aliyuncs.com/v1",
        "apiKey": "sk-sp-YOUR_API_KEY",
        "api": "openai-completions",
        "models": [
          {
            "id": "qwen3.5-plus",
            "name": "qwen3.5-plus",
            "api": "openai-completions",
            "reasoning": false,
            "input": ["text", "image"],
            "contextWindow": 1000000,
            "maxTokens": 65536
          },
          {
            "id": "qwen3-max-2026-01-23",
            "name": "qwen3-max-thinking",
            "api": "openai-completions",
            "reasoning": false,
            "input": ["text"],
            "contextWindow": 262144,
            "maxTokens": 65536
          },
          {
            "id": "qwen3-coder-next",
            "name": "qwen3-coder-next",
            "api": "openai-completions",
            "reasoning": false,
            "input": ["text"],
            "contextWindow": 262144,
            "maxTokens": 65536
          },
          {
            "id": "qwen3-coder-plus",
            "name": "qwen3-coder-plus",
            "api": "openai-completions",
            "reasoning": false,
            "input": ["text"],
            "contextWindow": 1000000,
            "maxTokens": 65536
          },
          {
            "id": "MiniMax-M2.5",
            "name": "MiniMax-M2.5",
            "api": "openai-completions",
            "reasoning": false,
            "input": ["text"],
            "contextWindow": 204800,
            "maxTokens": 131072
          },
          {
            "id": "glm-5",
            "name": "glm-5",
            "api": "openai-completions",
            "reasoning": false,
            "input": ["text"],
            "contextWindow": 202752,
            "maxTokens": 16384
          },
          {
            "id": "glm-4.7",
            "name": "glm-4.7",
            "api": "openai-completions",
            "reasoning": false,
            "input": ["text"],
            "contextWindow": 202752,
            "maxTokens": 16384
          },
          {
            "id": "kimi-k2.5",
            "name": "kimi-k2.5",
            "api": "openai-completions",
            "reasoning": false,
            "input": ["text", "image"],
            "contextWindow": 262144,
            "maxTokens": 32768
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": "bailian/qwen3.5-plus",
      "models": {
        "bailian/qwen3-max-2026-01-23": {},
        "bailian/qwen3.5-plus": {},
        "bailian/qwen3-coder-next": {},
        "bailian/qwen3-coder-plus": {},
        "bailian/MiniMax-M2.5": {},
        "bailian/glm-5": {},
        "bailian/glm-4.7": {},
        "bailian/kimi-k2.5": {}
      }
    },
    "list": [
      {
        "id": "main",
        "model": "bailian/qwen3.5-plus"
      }
    ]
  }
}
```

---

## 📊 模型对比表

| 模型 | 上下文长度 | 最大输出 | 多模态 | 推理 | 适用场景 |
|------|-----------|---------|--------|------|---------|
| **qwen3.5-plus** | 1,000,000 | 65,536 | ✅ | ❌ | 通用任务、长文档分析 |
| **qwen3-max-thinking** | 262,144 | 65,536 | ❌ | ❌ | 复杂推理、数学问题 |
| **qwen3-coder-next** | 262,144 | 65,536 | ❌ | ❌ | 代码生成、快速迭代 |
| **qwen3-coder-plus** | 1,000,000 | 65,536 | ❌ | ❌ | 大型代码库分析 |
| **MiniMax-M2.5** | 204,800 | 131,072 | ❌ | ❌ | 长文本生成 |
| **glm-5** | 202,752 | 16,384 | ❌ | ❌ | 通用对话、中文优化 |
| **glm-4.7** | 202,752 | 16,384 | ❌ | ❌ | 轻量级任务 |
| **kimi-k2.5** | 262,144 | 32,768 | ✅ | ❌ | 多模态理解、长文档 |

---

## 🎯 模型选型建议

### 按任务类型

| 任务类型 | 推荐模型 | 理由 |
|---------|---------|------|
| **日常对话** | `qwen3.5-plus` | 平衡性能与成本，支持图像 |
| **复杂推理** | `qwen3-max-thinking` | 最强推理能力 |
| **代码生成** | `qwen3-coder-next` | 快速响应，代码优化 |
| **代码审查** | `qwen3-coder-plus` | 超长上下文，完整项目分析 |
| **长文档分析** | `qwen3.5-plus` | 1M token 上下文 |
| **多模态任务** | `qwen3.5-plus` / `kimi-k2.5` | 支持图像输入 |
| **中文优化** | `glm-5` | 智谱中文能力优秀 |

### 按上下文长度需求

| 上下文需求 | 推荐模型 |
|-----------|---------|
| < 100K | `glm-4.7`, `glm-5` |
| 100K - 200K | `glm-5`, `MiniMax-M2.5` |
| 200K - 500K | `kimi-k2.5`, `qwen3-max-thinking` |
| > 500K | `qwen3.5-plus`, `qwen3-coder-plus` |
| 1M+ | `qwen3.5-plus` |

---

## 🔑 配置字段说明

### models.providers.bailian

| 字段 | 说明 | 必填 | 示例值 |
|------|------|------|--------|
| `baseUrl` | API 端点 URL | ✅ | `https://coding.dashscope.aliyuncs.com/v1` |
| `apiKey` | 百炼 API Key | ✅ | `sk-sp-xxxxxxxx` |
| `api` | API 类型 | ✅ | `openai-completions` |
| `models` | 模型列表 | ✅ | 见下表 |

### models[].model

| 字段 | 说明 | 必填 | 示例值 |
|------|------|------|--------|
| `id` | 模型 ID（与百炼一致） | ✅ | `qwen3.5-plus` |
| `name` | 显示名称 | ✅ | `qwen3.5-plus` |
| `api` | API 类型 | ✅ | `openai-completions` |
| `reasoning` | 是否推理模型 | ❌ | `false` |
| `input` | 支持的输入类型 | ❌ | `["text", "image"]` |
| `contextWindow` | 上下文窗口 (tokens) | ✅ | `1000000` |
| `maxTokens` | 最大输出 token 数 | ✅ | `65536` |
| `cost` | 成本配置 | ❌ | 见下文 |

### cost 字段（可选）

```json
"cost": {
  "input": 0,
  "output": 0,
  "cacheRead": 0,
  "cacheWrite": 0
}
```

用于跟踪 token 使用成本，单位为元/千 token。

---

## 📝 配置命令

### 方式一：CLI 命令配置

```bash
# 配置 bailian provider
docker exec stock node dist/index.js config set models.providers.bailian '{
  "baseUrl": "https://coding.dashscope.aliyuncs.com/v1",
  "apiKey": "sk-sp-YOUR_API_KEY",
  "api": "openai-completions",
  "models": [...]
}'

# 设置默认模型
docker exec stock node dist/index.js config set agents.defaults.model "bailian/qwen3.5-plus"

# 配置可用模型列表
docker exec stock node dist/index.js config set agents.defaults.models '{
  "bailian/qwen3.5-plus": {},
  "bailian/qwen3-max-2026-01-23": {},
  "bailian/qwen3-coder-plus": {}
}'
```

### 方式二：直接编辑 openclaw.json

```bash
# 编辑配置文件
code ~/.openclaw-stock/openclaw.json

# 重启容器应用配置
docker restart stock
```

---

## 🐛 常见问题

### 1. "Unknown model" 错误

**原因**: 模型未在 `models.providers.bailian.models` 中定义

**解决**: 确保模型配置包含在 providers 中，并重启容器

### 2. 上下文长度不生效

**原因**: `contextWindow` 配置错误或单位不对

**解决**: 确认使用 tokens 为单位（不是字符）

### 3. API Key 无效

**检查**:
```bash
# 验证 API Key 格式
docker exec stock node dist/index.js config get models.providers.bailian.apiKey
```

**解决**: 在百炼控制台重新生成 API Key

---

## 💡 最佳实践建议

### 1. 模型分级配置

```json
{
  "agents": {
    "defaults": {
      "model": "bailian/qwen3.5-plus",  // 默认使用
      "models": {
        "bailian/qwen3.5-plus": {},      // 常用
        "bailian/qwen3-max-2026-01-23": {}, // 复杂任务
        "bailian/qwen3-coder-plus": {}   // 代码任务
      }
    }
  }
}
```

### 2. 按 Agent 分配模型

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "model": "bailian/qwen3.5-plus"    // 通用任务
      },
      {
        "id": "coding",
        "model": "bailian/qwen3-coder-plus" // 代码任务
      },
      {
        "id": "analysis",
        "model": "bailian/qwen3.5-plus"    // 长文档分析
      }
    ]
  }
}
```

### 3. 成本优化

- 日常任务使用 `qwen3.5-plus`（性价比高）
- 复杂推理才用 `qwen3-max-thinking`（成本高）
- 代码任务用 `qwen3-coder-next`（快速响应）

### 4. 上下文管理

- 1M 上下文模型适合长文档分析
- 日常对话 200K 足够
- 注意 `maxTokens` 限制输出长度

---

## 🔗 相关资源

- **百炼控制台**: https://bailian.console.aliyun.com/
- **API 文档**: https://help.aliyun.com/zh/dashscope/
- **OpenClaw 文档**: https://docs.openclaw.ai/providers/qwen
- **模型价格**: https://help.aliyun.com/zh/dashscope/pricing

---

## 📚 变更记录

| 日期 | 版本 | 说明 |
|------|------|------|
| 2026-04-06 | 1.0 | 初始版本，记录完整配置实践 |

---

*本文档由 AI 生成并维护，如有问题请提交 PR 更新。*
