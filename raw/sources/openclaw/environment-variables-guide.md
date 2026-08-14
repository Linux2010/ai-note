# OpenClaw 环境变量完全指南

> **版本**: 1.0  
> **最后更新**: 2026-04-10  
> **适用版本**: OpenClaw 2026.4.2+  
> **来源**: 官方文档 + 生产环境实践

---

## 📖 概述

OpenClaw 从多个来源加载环境变量，遵循**不覆盖已有值**的原则。本文档提供完整的配置指南、最佳实践和故障排查方法。

---

## 🎯 加载优先级（从高到低）

```
┌─────────────────────────────────────────────────────────┐
│  1️⃣ 进程环境变量                                        │
│     父 shell/daemon 传递给 Gateway 进程                  │
│     （最高优先级，会覆盖其他来源）                        │
├─────────────────────────────────────────────────────────┤
│  2️⃣ 当前工作目录的 .env                                 │
│     dotenv 默认行为，不覆盖已有值                        │
├─────────────────────────────────────────────────────────┤
│  3️⃣ 全局 .env ⭐ 推荐                                   │
│     ~/.openclaw/.env                                    │
│     不覆盖已有值，所有实例共享                           │
├─────────────────────────────────────────────────────────┤
│  4️⃣ Config env 块                                       │
│     ~/.openclaw/openclaw.json 中的 env 配置             │
│     仅在变量缺失时应用                                   │
├─────────────────────────────────────────────────────────┤
│  5️⃣ Login-shell 导入（可选）                            │
│     env.shellEnv.enabled 或 OPENCLAW_LOAD_SHELL_ENV=1   │
│     仅导入缺失的预期键                                   │
└─────────────────────────────────────────────────────────┘
```

---

## ✅ 最佳实践：全局 .env 文件

### 位置
```bash
~/.openclaw/.env
```

### 优点
- ✅ **自动加载** - OpenClaw 启动时自动扫描
- ✅ **无需配置** - 不需要在 openclaw.json 中声明
- ✅ **所有实例共享** - Main Agent + 所有 Docker 容器
- ✅ **集中管理** - 所有环境变量在一处
- ✅ **易于备份** - 单个文件备份即可

### 示例配置

```bash
# ========================================
# OpenClaw 全局环境变量配置
# ========================================

# ========================================
# AI Model Providers
# ========================================

### Bailian (阿里百炼)
BAILIAN_API_KEY=sk-sp-xxxxxxxxxxxx
BAILIAN_BASE_URL=https://coding.dashscope.aliyuncs.com/v1

### OpenAI
OPENAI_API_KEY=sk-xxxxxxxxxxxx

### Gemini
GEMINI_API_KEY=xxxxxxxxxxxx

### Ollama (本地)
OLLAMA_BASE_URL=http://127.0.0.1:11434/v1
OLLAMA_API_KEY=ollama-local

# ========================================
# Telegram Bot Tokens
# ========================================

### Main/Core Bot
TELEGRAM_CORE_BOT_TOKEN=8524712381:AAGZ...
TELEGRAM_CORE_BOT_USERNAME=@your_core_bot

### Coding Bot
TELEGRAM_CODING_BOT_TOKEN=8766121399:AAFy...
TELEGRAM_CODING_BOT_USERNAME=@your_coding_bot

### Stock Bot
TELEGRAM_STOCK_BOT_TOKEN=8295117727:AAEe...
TELEGRAM_STOCK_BOT_USERNAME=@your_stock_bot

### Hermes Bot
TELEGRAM_HERMES_BOT_TOKEN=8782809595:AAGL...
TELEGRAM_HERMES_BOT_USERNAME=@your_hermes_bot

# ========================================
# GitHub Tokens
# ========================================

### GitHub Personal Access Token
GH_TOKEN=gho_xxxxxxxxxxxxxxxxxxxx
GITHUB_API_TOKEN=gho_xxxxxxxxxxxxxxxxxxxx

# ========================================
# Network Proxy
# ========================================

### HTTP/HTTPS Proxy
HTTP_PROXY=http://127.0.0.1:7897
HTTPS_PROXY=http://127.0.0.1:7897
NO_PROXY=localhost,127.0.0.1,::1

# ========================================
# OpenClaw Runtime Configuration
# ========================================

### Logging
OPENCLAW_LOG_LEVEL=info

### UI Theme
OPENCLAW_THEME=dark

### Shell Environment Import
OPENCLAW_LOAD_SHELL_ENV=0

# ========================================
# Path Overrides (Optional)
# ========================================

# OPENCLAW_HOME=/path/to/custom/home
# OPENCLAW_STATE_DIR=/path/to/custom/state
# OPENCLAW_CONFIG_PATH=/path/to/custom/config.json
```

### 权限设置
```bash
# 设置安全权限（仅所有者可读写）
chmod 600 ~/.openclaw/.env

# 验证权限
ls -la ~/.openclaw/.env
# 输出：-rw-------  1 user  staff  ...  ~/.openclaw/.env
```

---

## 🔧 配置方法对比

### 方法 1：全局 .env（推荐⭐⭐⭐⭐⭐）

**适用场景**：生产环境、多容器部署

**配置**：
```bash
# 文件位置
~/.openclaw/.env

# 无需任何额外配置，OpenClaw 自动加载
```

**优点**：
- 自动加载
- 所有实例共享
- 集中管理
- 易于维护

---

### 方法 2：Config env 块（⭐⭐⭐）

**适用场景**：少量环境变量、配置集中管理

**配置**：
```json
{
  "env": {
    "OPENROUTER_API_KEY": "sk-or-...",
    "vars": {
      "GROQ_API_KEY": "gsk-..."
    }
  }
}
```

**位置**：`~/.openclaw/openclaw.json`

**优点**：
- 配置集中
- 支持 `${VAR}` 引用

**缺点**：
- 不适合大量变量
- JSON 格式不如 .env 简洁

---

### 方法 3：Shell 环境变量（⭐⭐）

**适用场景**：开发调试、临时测试

**配置**：
```json
{
  "env": {
    "shellEnv": {
      "enabled": true,
      "timeoutMs": 15000
    }
  }
}
```

**环境变量**：
```bash
export OPENCLAW_LOAD_SHELL_ENV=1
export OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000
```

**优点**：
- 复用现有 shell 配置
- 适合开发环境

**缺点**：
- Gateway 作为服务运行时可能无法继承
- 仅导入缺失的键

---

## 📋 环境变量分类参考

### AI Provider 配置

| 变量名 | 说明 | 示例 |
|--------|------|------|
| `BAILIAN_API_KEY` | 阿里百炼 API Key | `sk-sp-xxxxxxxxxxxx` |
| `OPENAI_API_KEY` | OpenAI API Key | `sk-xxxxxxxxxxxx` |
| `GEMINI_API_KEY` | Google Gemini API Key | `xxxxxxxxxxxx` |
| `OLLAMA_BASE_URL` | Ollama 服务地址 | `http://127.0.0.1:11434/v1` |

### Telegram Bot Tokens

| 变量名 | 说明 | 示例 |
|--------|------|------|
| `TELEGRAM_CORE_BOT_TOKEN` | Main Agent Bot Token | `8524712381:AAGZ...` |
| `TELEGRAM_CODING_BOT_TOKEN` | Coding Agent Bot Token | `8766121399:AAFy...` |
| `TELEGRAM_STOCK_BOT_TOKEN` | Stock Agent Bot Token | `8295117727:AAEe...` |

### GitHub Tokens

| 变量名 | 说明 | 示例 |
|--------|------|------|
| `GH_TOKEN` | GitHub CLI Token | `gho_xxxxxxxxxx` |
| `GITHUB_API_TOKEN` | GitHub API Token | `gho_xxxxxxxxxx` |

### 网络代理

| 变量名 | 说明 | 示例 |
|--------|------|------|
| `HTTP_PROXY` | HTTP 代理地址 | `http://127.0.0.1:7897` |
| `HTTPS_PROXY` | HTTPS 代理地址 | `http://127.0.0.1:7897` |
| `NO_PROXY` | 不走代理的地址 | `localhost,127.0.0.1` |

### OpenClaw 运行时

| 变量名 | 说明 | 默认值 |
|--------|------|--------|
| `OPENCLAW_LOG_LEVEL` | 日志级别 | `info` |
| `OPENCLAW_THEME` | UI 主题 | `auto` |
| `OPENCLAW_LOAD_SHELL_ENV` | 是否加载 shell 环境变量 | `0` |

### 路径覆盖（高级）

| 变量名 | 说明 | 示例 |
|--------|------|------|
| `OPENCLAW_HOME` | 覆盖 OpenClaw 主目录 | `/home/openclaw` |
| `OPENCLAW_STATE_DIR` | 覆盖状态目录 | `/var/lib/openclaw` |
| `OPENCLAW_CONFIG_PATH` | 覆盖配置文件路径 | `/etc/openclaw/config.json` |

---

## 🚀 快速开始

### 步骤 1：创建 .env 文件
```bash
# 创建目录（如果不存在）
mkdir -p ~/.openclaw

# 创建 .env 文件
cat > ~/.openclaw/.env << 'EOF'
# AI Provider
BAILIAN_API_KEY=sk-sp-xxxxxxxxxxxx

# Telegram
TELEGRAM_CORE_BOT_TOKEN=8524712381:AAGZ...

# GitHub
GH_TOKEN=gho_xxxxxxxxxxxxxxxxxxxx

# Proxy
HTTP_PROXY=http://127.0.0.1:7897
HTTPS_PROXY=http://127.0.0.1:7897
EOF

# 设置权限
chmod 600 ~/.openclaw/.env
```

### 步骤 2：验证配置
```bash
# 检查文件是否存在
ls -la ~/.openclaw/.env

# 检查内容
cat ~/.openclaw/.env | head -20
```

### 步骤 3：重启 Gateway
```bash
# 原生 Gateway
openclaw gateway restart

# Docker 容器
docker restart coding
docker restart stock
docker restart hermes
```

### 步骤 4：测试功能
```bash
# 检查环境变量是否加载
docker exec coding env | grep -E "API_KEY|TOKEN"

# 在 Telegram 中发送测试消息
# 检查 Bot 是否响应
```

---

## 🐛 故障排查

### 问题 1：环境变量未生效

**症状**：API 调用失败，提示缺少 Token

**排查步骤**：
```bash
# 1. 检查 .env 文件是否存在
ls -la ~/.openclaw/.env

# 2. 检查文件内容
cat ~/.openclaw/.env | grep API_KEY

# 3. 检查文件权限
ls -la ~/.openclaw/.env
# 应该是 -rw------- (600)

# 4. 重启 Gateway
openclaw gateway restart

# 5. 检查容器环境变量
docker exec coding env | grep API_KEY
```

**解决方案**：
- 确保 `.env` 文件格式正确（`KEY=VALUE`，无空格）
- 确保文件权限为 `600`
- 重启 Gateway 或容器

---

### 问题 2：容器无法访问 .env

**症状**：原生 Gateway 正常，Docker 容器报错

**排查步骤**：
```bash
# 1. 检查容器挂载
docker inspect coding --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}'

# 2. 验证 .env 在挂载目录中
ls -la ~/.openclaw/.env

# 3. 检查容器内 .env
docker exec coding ls -la ~/.openclaw/.env
```

**解决方案**：
- 确保容器挂载了 `~/.openclaw/` 目录
- 重启容器使挂载生效

---

### 问题 3：删除了 .env 文件

**症状**：所有 API 调用失败

**解决方案**：
```bash
# 1. 从备份恢复
cp ~/.env.backup ~/.openclaw/.env

# 2. 如果没有备份，重新创建
cat > ~/.openclaw/.env << 'EOF'
# 重新填入所有环境变量
BAILIAN_API_KEY=sk-sp-xxxxxxxxxxxx
TELEGRAM_CORE_BOT_TOKEN=...
GH_TOKEN=gho_xxxxxxxxxx
EOF

# 3. 设置权限
chmod 600 ~/.openclaw/.env

# 4. 重启所有服务
openclaw gateway restart
docker restart coding
docker restart stock
docker restart hermes
```

---

## ⚠️ 安全注意事项

### 1. 文件权限
```bash
# ✅ 正确：仅所有者可读写
chmod 600 ~/.openclaw/.env

# ❌ 错误：其他人可读
chmod 644 ~/.openclaw/.env
```

### 2. 不要提交到 Git
```bash
# 在 .gitignore 中添加
echo ".env" >> ~/.openclaw/.gitignore
```

### 3. 定期备份
```bash
# 定期备份 .env 文件
cp ~/.openclaw/.env ~/.openclaw/.env.backup.$(date +%Y%m%d)

# 备份到安全位置（如加密存储）
```

### 4. 使用 SecretRef（生产环境）
```json
{
  "models": {
    "providers": {
      "openai": {
        "apiKey": {
          "source": "env",
          "provider": "default",
          "id": "OPENAI_API_KEY"
        }
      }
    }
  }
}
```

---

## 📊 容器部署特殊说明

### Docker 容器环境变量继承

```yaml
# docker-compose.yml 示例
services:
  coding:
    image: ghcr.io/openclaw/openclaw:2026.4.2
    volumes:
      - ~/.openclaw-coding:/home/node/.openclaw
    environment:
      # 可选：显式传递环境变量
      - TELEGRAM_BOT_TOKEN=${TELEGRAM_CODING_BOT_TOKEN}
```

### 挂载共享 .env

```bash
# 所有容器共享同一个 .env 文件
~/.openclaw/.env
    ↓ 挂载
/home/node/.openclaw/.env (容器内)
```

### 验证容器环境变量

```bash
# 检查容器环境变量
docker exec coding env | grep -E "API_KEY|TOKEN"

# 查看容器日志
docker logs coding 2>&1 | grep -i "telegram\|provider"
```

---

## 🎯 最佳实践总结

### ✅ 推荐做法

1. **使用全局 .env** - `~/.openclaw/.env`
2. **设置正确权限** - `chmod 600`
3. **定期备份** - 保留多个备份副本
4. **集中管理** - 所有环境变量在一处
5. **不要提交到 Git** - 添加到 `.gitignore`

### ❌ 避免做法

1. **不要删除 .env** - 会导致 API 调用失败
2. **不要设置宽松权限** - 避免 `chmod 777`
3. **不要硬编码在配置中** - 使用环境变量引用
4. **不要共享敏感信息** - 保护 API Key 和 Token

---

## 🔗 相关资源

### 官方文档
- [Environment Variables](https://docs.openclaw.ai/help/environment)
- [Configuration Reference](https://docs.openclaw.ai/gateway/configuration-reference)
- [Secrets Management](https://docs.openclaw.ai/gateway/secrets)

### 相关技能
- `openclaw-guide` - OpenClaw 官方指南
- `cron-scheduler` - 定时任务配置
- `memory-core` - Memory 配置

---

## 📝 版本历史

| 版本 | 日期 | 更新内容 |
|------|------|---------|
| 1.0 | 2026-04-10 | 初始版本，基于官方文档 + 生产实践 |

---

*本文档遵循 MIT 许可证。*
*基于 OpenClaw 官方文档和真实生产环境部署经验编写。*
