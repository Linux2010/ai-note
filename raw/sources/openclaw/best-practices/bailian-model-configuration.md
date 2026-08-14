# OpenClaw 模型配置指南 - 百炼 Qwen

**版本**: 2026.4.2  
**创建时间**: 2026-04-06  
**更新**: 2026-04-06 (修正：在 openclaw.json 中配置 providers)  
**标签**: `models` `providers` `bailian` `qwen` `configuration`

---

## 📋 概述

本文档说明如何在 OpenClaw 中配置阿里云百炼（Qwen）模型提供商。

**关键点**: 模型提供商配置应放在 **`openclaw.json`** 的 `models.providers` 字段中，而不是单独的 `models.json` 文件。

---

## 🔧 配置方式

### 方式一：通过 CLI 命令（推荐）

```bash
# 配置 bailian provider
docker exec stock node dist/index.js config set models.providers.bailian '{
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
    }
  ]
}'
```

### 方式二：直接编辑 openclaw.json

```json5
{
  // ... 其他配置 ...
  
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
          }
        ]
      }
    }
  },
  
  "agents": {
    "defaults": {
      "model": "bailian/qwen3.5-plus"
    }
  }
}
```

---

## 📁 配置文件位置

```
~/.openclaw/
├── openclaw.json          # ✅ 全局配置（包含 models.providers）
└── agents/
    └── main/
        └── agent/
            ├── models.json        # ❌ 已过时（旧版本兼容）
            └── auth-profiles.json # ⚠️ 认证配置文件
```

**重要**: 
- `models.providers` 配置在 `openclaw.json` 中
- API Key 可以放在 `openclaw.json` 或 `auth-profiles.json` 中

---

## 🔑 获取百炼 API Key

1. 访问：https://bailian.console.aliyun.com/
2. 登录阿里云账号
3. 进入 **API-KEY 管理**
4. 创建或复制现有 API Key

---

## 🎯 配置字段说明

### models.providers.bailian

| 字段 | 说明 | 示例值 |
|------|------|--------|
| `baseUrl` | API 端点 URL | `https://coding.dashscope.aliyuncs.com/v1` |
| `apiKey` | 百炼 API Key | `sk-sp-xxxxxxxx` |
| `api` | API 类型 | `openai-completions` |
| `models` | 模型列表 | 见下表 |

### models[].model

| 字段 | 说明 | 示例值 |
|------|------|--------|
| `id` | 模型 ID（与百炼一致） | `qwen3.5-plus` |
| `name` | 显示名称 | `qwen3.5-plus` |
| `api` | API 类型 | `openai-completions` |
| `reasoning` | 是否推理模型 | `false` |
| `input` | 支持的输入类型 | `["text", "image"]` |
| `contextWindow` | 上下文窗口 | `1000000` |
| `maxTokens` | 最大输出 token 数 | `65536` |

---

## ✅ 验证配置

### 1. 检查配置文件

```bash
docker exec stock cat /home/node/.openclaw/openclaw.json
```

### 2. 查看日志确认模型加载

```bash
docker logs stock --tail 20 | grep "agent model"
```

**预期输出**:
```
agent model: bailian/qwen3.5-plus
```

### 3. 测试对话

访问 http://localhost:9999/ 并发送测试消息。

---

## 🐛 常见问题

### 1. "Unknown model: bailian/qwen3.5-plus"

**原因**: `models.providers` 未配置或配置错误

**解决**:
```bash
docker exec stock node dist/index.js config set models.providers.bailian '{...}'
docker restart stock
```

### 2. "No API key found for provider bailian"

**原因**: API Key 未配置或 auth-profiles.json 缺失

**解决方式 A** - 在 openclaw.json 中配置:
```json
{
  "models": {
    "providers": {
      "bailian": {
        "apiKey": "sk-sp-xxx"
      }
    }
  }
}
```

**解决方式 B** - 创建 auth-profiles.json:
```bash
docker exec stock mkdir -p /home/node/.openclaw/agents/main/agent
docker exec stock cat > /home/node/.openclaw/agents/main/agent/auth-profiles.json << 'EOF'
{
  "bailian": {
    "apiKey": "sk-sp-xxx"
  }
}
EOF
```

### 3. 配置后仍无法使用

**检查步骤**:
```bash
# 1. 确认配置已保存
docker exec stock cat /home/node/.openclaw/openclaw.json | grep -A 20 "models"

# 2. 重启容器
docker restart stock

# 3. 查看完整日志
docker logs stock --tail 50
```

---

## 📚 官方文档参考

- [Model Providers](https://docs.openclaw.ai/concepts/model-providers)
- [Configuration Reference](https://docs.openclaw.ai/gateway/configuration-reference)
- [Qwen Provider](https://docs.openclaw.ai/providers/qwen)

---

## 🔄 变更记录

| 日期 | 版本 | 说明 |
|------|------|------|
| 2026-04-06 | 1.1 | 修正：配置应在 openclaw.json 而非 models.json |
| 2026-04-06 | 1.0 | 初始版本 |

---

*本文档由 AI 生成并维护，如有问题请提交 PR 更新。*
