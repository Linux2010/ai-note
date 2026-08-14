# 阿里云百炼 Coding Plan 完整配置指南

**版本**: 2026.4.15  
**创建时间**: 2026-04-15  
**平台**: 阿里云百炼 Coding Plan  
**标签**: `bailian` `coding-plan` `models` `configuration` `best-practices`

---

## 📋 概述

本文档记录阿里云百炼 Coding Plan 在 OpenClaw 中的完整配置实践，包含 9 个官方支持模型的准确参数、常见踩坑和验证方法。

**重要**: Coding Plan 使用专属 API 端点 `https://coding.dashscope.aliyuncs.com/v1`，与标准百炼 API 不同。

---

## 🎯 支持的模型清单 (共 9 个)

| 模型 ID | 显示名称 | Context Window | Max Tokens | 输入类型 | Thinking 支持 |
|---------|---------|---------------|-----------|---------|--------------|
| `qwen3.6-plus` | qwen3.6-plus | 1,000,000 | 65,536 | text, image | ✅ Qwen |
| `qwen3.5-plus` | qwen3.5-plus | 1,000,000 | 65,536 | text, image | ✅ Qwen |
| `qwen3-max-2026-01-23` | qwen3-max-thinking | 262,144 | 65,536 | text | ✅ Qwen |
| `qwen3-coder-next` | qwen3-coder-next | 262,144 | 65,536 | text | - |
| `qwen3-coder-plus` | qwen3-coder-plus | 1,000,000 | 65,536 | text | - |
| `MiniMax-M2.5` | MiniMax-M2.5 | 196,608 | 32,768 | text | - |
| `glm-5` | glm-5 | 202,752 | 16,384 | text | ✅ Qwen |
| `glm-4.7` | glm-4.7 | 202,752 | 16,384 | text | ✅ Qwen |
| `kimi-k2.5` | kimi-k2.5 | 262,144 | 32,768 | text, image | ✅ Qwen |

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
            "id": "qwen3.6-plus",
            "name": "qwen3.6-plus",
            "api": "openai-completions",
            "reasoning": false,
            "input": ["text", "image"],
            "cost": {"input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0},
            "contextWindow": 1000000,
            "maxTokens": 65536,
            "compat": {"thinkingFormat": "qwen"}
          },
          {
            "id": "qwen3.5-plus",
            "name": "qwen3.5-plus",
            "api": "openai-completions",
            "reasoning": false,
            "input": ["text", "image"],
            "cost": {"input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0},
            "contextWindow": 1000000,
            "maxTokens": 65536,
            "compat": {"thinkingFormat": "qwen"}
          },
          {
            "id": "qwen3-max-2026-01-23",
            "name": "qwen3-max-thinking",
            "api": "openai-completions",
            "reasoning": false,
            "input": ["text"],
            "cost": {"input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0},
            "contextWindow": 262144,
            "maxTokens": 65536,
            "compat": {"thinkingFormat": "qwen"}
          },
          {
            "id": "qwen3-coder-next",
            "name": "qwen3-coder-next",
            "api": "openai-completions",
            "reasoning": false,
            "input": ["text"],
            "cost": {"input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0},
            "contextWindow": 262144,
            "maxTokens": 65536
          },
          {
            "id": "qwen3-coder-plus",
            "name": "qwen3-coder-plus",
            "api": "openai-completions",
            "reasoning": false,
            "input": ["text"],
            "cost": {"input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0},
            "contextWindow": 1000000,
            "maxTokens": 65536
          },
          {
            "id": "MiniMax-M2.5",
            "name": "MiniMax-M2.5",
            "api": "openai-completions",
            "reasoning": false,
            "input": ["text"],
            "cost": {"input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0},
            "contextWindow": 196608,
            "maxTokens": 32768
          },
          {
            "id": "glm-5",
            "name": "glm-5",
            "api": "openai-completions",
            "reasoning": false,
            "input": ["text"],
            "cost": {"input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0},
            "contextWindow": 202752,
            "maxTokens": 16384,
            "compat": {"thinkingFormat": "qwen"}
          },
          {
            "id": "glm-4.7",
            "name": "glm-4.7",
            "api": "openai-completions",
            "reasoning": false,
            "input": ["text"],
            "cost": {"input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0},
            "contextWindow": 202752,
            "maxTokens": 16384,
            "compat": {"thinkingFormat": "qwen"}
          },
          {
            "id": "kimi-k2.5",
            "name": "kimi-k2.5",
            "api": "openai-completions",
            "reasoning": false,
            "input": ["text", "image"],
            "cost": {"input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0},
            "contextWindow": 262144,
            "maxTokens": 32768,
            "compat": {"thinkingFormat": "qwen"}
          }
        ]
      },
      "ollama": {
        "baseUrl": "http://host.docker.internal:11434/v1",
        "apiKey": "ollama-local",
        "api": "openai-completions",
        "models": [
          {
            "id": "nomic-embed-text",
            "name": "nomic-embed-text",
            "api": "openai-completions"
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "bailian/qwen3.6-plus"
      },
      "models": {
        "bailian/qwen3.6-plus": {},
        "bailian/qwen3-max-2026-01-23": {"alias": "qwen3-max-thinking"},
        "bailian/qwen3.5-plus": {},
        "bailian/qwen3-coder-next": {},
        "bailian/qwen3-coder-plus": {},
        "bailian/MiniMax-M2.5": {},
        "bailian/glm-5": {},
        "bailian/glm-4.7": {},
        "bailian/kimi-k2.5": {}
      },
      "maxConcurrent": 4,
      "subagents": {"maxConcurrent": 8}
    }
  }
}
```

---

## ⚠️ 常见踩坑与解决方案

### 1. MiniMax-M2.5 参数易错 ❌❌❌

**错误配置**:
```json
{
  "contextWindow": 204800,  // ❌ 错误
  "maxTokens": 131072       // ❌ 错误
}
```

**正确配置**:
```json
{
  "contextWindow": 196608,  // ✅ 正确
  "maxTokens": 32768        // ✅ 正确
}
```

**原因**: 官方文档参数与实际 API 不符，以实际测试为准。

---

### 2. Thinking 模式支持配置

以下 **6 个模型** 支持深度思考，必须添加 `compat.thinkingFormat` 字段：

```json
"compat": {"thinkingFormat": "qwen"}
```

**需要配置的模型**:
- ✅ `qwen3.6-plus`
- ✅ `qwen3.5-plus`
- ✅ `qwen3-max-2026-01-23`
- ✅ `glm-5`
- ✅ `glm-4.7`
- ✅ `kimi-k2.5`

**不需要配置的模型**:
- `qwen3-coder-next`
- `qwen3-coder-plus`
- `MiniMax-M2.5`

---

### 3. agents.defaults.models 必须与 providers 同步

**错误**: 只在 `providers.bailian.models` 中定义模型，忘记在 `agents.defaults.models` 添加

**后果**: 模型列表不显示，无法选择使用

**正确做法**: 每个在 `providers` 中定义的模型，都必须在 `agents.defaults.models` 中有对应条目

```json
{
  "models": {
    "providers": {
      "bailian": {
        "models": [
          {"id": "qwen3.6-plus", ...}
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "models": {
        "bailian/qwen3.6-plus": {}  // ✅ 必须同步
      }
    }
  }
}
```

---

### 4. OpenClaw 不支持 video 输入

虽然 `qwen3.6-plus` 本身支持视频理解，但 OpenClaw 配置中 `input` 字段只支持：
- `["text"]`
- `["text", "image"]`

**不能添加** `"video"`，会导致配置解析失败。

---

## 📝 多容器环境配置实践

### 容器列表

| 容器名 | 端口 | 职责 | 推荐默认模型 |
|--------|------|------|-------------|
| `hermes` | 9996 | Hermes 项目 AI 编程 | bailian/glm-5 |
| `coding` | 9997 | GitHub 工作流、PR 管理 | bailian/glm-5 |
| `alone` | 9998 | 独立任务/测试环境 | bailian/qwen3.6-plus |
| `stock` | 9999 | 股票监控、市场分析 | bailian/qwen3.6-plus |

### Docker 环境特殊配置

```json
{
  "models": {
    "providers": {
      "ollama": {
        "baseUrl": "http://host.docker.internal:11434/v1"  // Docker 内访问宿主机
      }
    }
  },
  "channels": {
    "telegram": {
      "proxy": "http://host.docker.internal:7897"  // Docker 内访问宿主机代理
    }
  }
}
```

---

## ✅ 验证命令

```bash
# 主实例
openclaw models list

# Coding 容器
docker exec coding openclaw models list

# 检查模型配置数量
cat ~/.openclaw/openclaw.json | jq '.models.providers.bailian.models | length'

# 验证默认模型
docker exec coding openclaw models list | grep default
```

**预期输出示例**:
```
Model                                      Input      Ctx      Local Auth  Tags
bailian/qwen3.6-plus                       text+image 977k     no    yes   default,configured
bailian/qwen3-max-2026-01-23               text       256k     no    yes   configured,alias:qwen3-max-thinking
bailian/qwen3.5-plus                       text+image 977k     no    yes   configured
...
```

---

## 🔄 生效操作

```bash
# 主实例
openclaw gateway restart

# Coding 容器
docker restart coding

# 所有容器
docker restart hermes coding alone stock
```

---

## 📊 套餐说明

| 套餐 | 支持模型 | 说明 |
|------|---------|------|
| **Lite** | 除 qwen3.6-plus 外的所有模型 | 已停止续费 |
| **Pro** | 全部 9 个模型 | 含 qwen3.6-plus 专属 |

---

## 💡 最佳实践建议

### 1. 模型分级使用

| 任务类型 | 推荐模型 | 理由 |
|---------|---------|------|
| 日常对话 | `qwen3.5-plus` / `glm-5` | 平衡性能与响应速度 |
| 复杂推理 | `qwen3-max-thinking` | 最强推理能力 |
| 代码生成 | `qwen3-coder-next` | 快速响应，代码优化 |
| 代码审查 | `qwen3-coder-plus` | 超长上下文，完整项目分析 |
| 长文档分析 | `qwen3.6-plus` | 1M token 上下文 |
| 多模态任务 | `qwen3.6-plus` / `kimi-k2.5` | 支持图像输入 |

### 2. 配置管理

- 将完整配置保存在版本控制中（如 GitHub）
- 使用 `jq` 工具验证 JSON 格式
- 修改配置后备份原文件
- 重启容器后验证模型列表

### 3. 成本优化

- 日常任务使用 `glm-5` 或 `qwen3.5-plus`
- 复杂推理才用 `qwen3-max-thinking`
- 代码任务用 `qwen3-coder-next`（快速响应）

---

## 🔗 相关资源

- **百炼 Coding Plan 控制台**: https://bailian.console.aliyun.com/
- **官方文档**: https://help.aliyun.com/zh/model-studio/openclaw-coding-plan
- **OpenClaw 文档**: https://docs.openclaw.ai/providers/qwen
- **模型价格**: https://help.aliyun.com/zh/dashscope/pricing

---

## 📚 变更记录

| 日期 | 版本 | 说明 |
|------|------|------|
| 2026-04-15 | 1.0 | 初始版本，记录 Coding Plan 完整配置和踩坑经验 |

---

*本文档由 AI 生成并维护，基于实际生产环境配置经验总结。*
