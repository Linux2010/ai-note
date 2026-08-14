# OpenClaw Docker 快速部署指南

**版本**: 2026.4.2  
**创建时间**: 2026-04-06  
**更新时间**: 2026-04-06 (修正挂载目录)  
**标签**: docker, deployment, quickstart, configuration

---

## 📋 概述

本文档记录使用 Docker 快速部署 OpenClaw 容器的完整流程，包括关键配置步骤和常见问题解决方案。

**⚠️ 重要**：挂载目录必须是 `/home/node/.openclaw`（不是 `/root/.openclaw`）！

---

## 🚀 快速启动

### 方案一：Docker Run（推荐）

```bash
# 一键启动命令（注意挂载目录！）
docker run -d \
  --name stock \
  -p 9999:18789 \
  -v ~/.openclaw-stock:/home/node/.openclaw \
  -e OPENCLAW_GATEWAY_PORT=18789 \
  ghcr.io/openclaw/openclaw:2026.4.2
```

### 方案二：Docker Compose

```yaml
version: '3.8'
services:
  openclaw:
    image: ghcr.io/openclaw/openclaw:2026.4.2
    container_name: stock
    ports:
      - "9999:18789"
    volumes:
      - ~/.openclaw-stock:/home/node/.openclaw
    environment:
      - OPENCLAW_GATEWAY_PORT=18789
    restart: unless-stopped
```

---

## ⚠️ 关键配置（必须执行）

### 步骤 1: ⚠️ 正确的挂载目录（关键！）

**容器内路径**: `/home/node/.openclaw`（不是 `/root/.openclaw`）

```bash
# ❌ 错误：挂载到 /root/.openclaw
docker run -v ~/.openclaw-stock:/root/.openclaw ...

# ✅ 正确：挂载到 /home/node/.openclaw
docker run -v ~/.openclaw-stock:/home/node/.openclaw ...
```

**原因**：容器以 `node` 用户运行，数据目录在 `/home/node/.openclaw`，挂载错误会导致数据不同步！

### 步骤 2: 配置绑定模式

```bash
docker exec stock node dist/index.js config set gateway.bind "lan"
```

**原因**: 默认绑定到 `127.0.0.1`（容器内部），设置为 `lan` 后绑定到 `0.0.0.0`，允许宿主机访问。

### 步骤 3: 配置允许的源地址

```bash
docker exec stock node dist/index.js config set gateway.controlUi.allowedOrigins '["http://localhost:9999","http://127.0.0.1:9999"]'
```

**原因**: 防止浏览器访问时报 "origin not allowed" 错误。

### 步骤 4: 设置认证 Token

```bash
docker exec stock node dist/index.js config set gateway.auth.token "hope"
```

### 步骤 5: 配置模型 Provider（百炼 Qwen）

```bash
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
    }
  ]
}'
```

### 步骤 6: 设置默认模型

```bash
docker exec stock node dist/index.js config set agents.defaults.model "bailian/qwen3.5-plus"
```

### 步骤 7: 重启容器

```bash
docker restart stock
```

---

## ✅ 验证步骤

### 检查容器状态

```bash
docker ps | grep stock
```

### 检查挂载配置

```bash
# 查看挂载配置
docker inspect stock | grep -A 5 "Mounts"

# 预期输出：
# "Source": "/Users/hope/.openclaw-stock"
# "Destination": "/home/node/.openclaw"
```

### 验证数据同步

```bash
# 查看容器内数据
docker exec stock ls -la /home/node/.openclaw/

# 查看宿主机数据
ls -la ~/.openclaw-stock/

# 两者应该包含相同的文件
```

### 检查端口映射

```bash
docker port stock
```

### 检查日志（确认监听 0.0.0.0）

```bash
docker logs stock --tail 20
```

**正确输出示例**:
```
listening on ws://0.0.0.0:18789
agent model: bailian/qwen3.5-plus
```

### 测试连接

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:9999/
# 应返回 200
```

---

## 🔑 访问信息

| 项目 | 值 |
|------|-----|
| 访问地址 | http://localhost:9999/ |
| Token | hope |
| 容器名称 | stock |
| 数据卷 | ~/.openclaw-stock |
| 容器内路径 | `/home/node/.openclaw` ⚠️ |
| 内部端口 | 18789 |
| 外部端口 | 9999 |

---

## 🐛 常见问题

### 1. "origin not allowed" 错误

**症状**: 浏览器控制台显示 `origin not allowed (open the Control UI from the gateway host or allow it in gateway.controlUi.allowedOrigins)`

**解决**:
```bash
docker exec stock node dist/index.js config set gateway.controlUi.allowedOrigins '["http://localhost:9999","http://127.0.0.1:9999"]'
docker restart stock
```

### 2. 挂载目录错误导致数据不同步

**症状**: `~/.openclaw-stock` 目录是空的，或者容器重启后配置丢失

**检查**:
```bash
# 查看挂载配置
docker inspect stock | grep -A 5 "Mounts"

# 查看容器内实际数据目录
docker exec stock ls -la /home/node/.openclaw/

# 查看宿主机目录
ls -la ~/.openclaw-stock/
```

**解决**: 确保挂载到 `/home/node/.openclaw` 而不是 `/root/.openclaw`

### 3. "pairing required" 错误

**症状**: 登录后仍提示需要配对

**解决**:
```bash
# 查看待处理设备
docker exec stock node dist/index.js devices list

# 批准设备（替换 <request-id> 为实际 ID）
docker exec stock node dist/index.js devices approve <request-id>
```

### 4. 无法访问 9999 端口

**检查步骤**:
```bash
# 检查端口是否被占用
lsof -i :9999

# 检查容器状态
docker ps | grep stock

# 检查网关绑定
docker exec stock node dist/index.js config get gateway.bind
# 应返回 "lan"
```

### 5. 容器启动后自动退出

**检查日志**:
```bash
docker logs stock --tail 50
```

**常见原因**:
- 配置文件格式错误
- 端口冲突
- 权限问题

---

## 🔧 高级技巧

### 技巧 1：禁用 Docker 全局代理

**问题**: Docker Desktop 设置了全局代理（`~/.docker/config.json` 中 `proxies.default`），新容器自动继承代理环境变量，但代理不可用会导致网络失败。

**症状**:
```
[telegram] deleteWebhook failed: Network request for 'deleteWebhook' failed!
```

**解决方案**: 创建容器时显式禁用代理

```bash
docker run -d --name stock \
  -p 9999:18789 \
  -v ~/.openclaw-stock:/home/node/.openclaw \
  -e OPENCLAW_GATEWAY_PORT=18789 \
  -e HTTP_PROXY= \
  -e HTTPS_PROXY= \
  -e http_proxy= \
  -e https_proxy= \
  -e NO_PROXY= \
  -e no_proxy= \
  ghcr.io/openclaw/openclaw:latest
```

**验证**:
```bash
docker exec stock curl -s https://api.telegram.org
# 应返回 JSON 响应
```

---

### 技巧 2：快速安装 Playwright（容器复制法）

**场景**: 容器内安装 Playwright + Chromium 很慢（网络问题、代理问题）

**解决方案**: 从已安装的容器复制文件

```bash
# 源容器（已安装 Playwright）
SOURCE=openclaw-trump
TARGET=openclaw-anna

# 1. 打包 Python 包
docker exec $SOURCE bash -c "
  cd ~/.local/lib/python3.11/site-packages
  tar czf /tmp/pw.tar.gz playwright pyee greenlet
"

# 2. 打包 Chromium
docker exec $SOURCE bash -c "
  cd ~/.cache && tar czf /tmp/chromium.tar.gz ms-playwright
"

# 3. 复制到目标容器
docker cp $SOURCE:/tmp/pw.tar.gz /tmp/
docker cp $SOURCE:/tmp/chromium.tar.gz /tmp/
docker cp /tmp/pw.tar.gz $TARGET:/tmp/
docker cp /tmp/chromium.tar.gz $TARGET:/tmp/

# 4. 解压
docker exec $TARGET bash -c "
  cd ~/.local/lib/python3.11/site-packages
  tar xzf /tmp/pw.tar.gz
"
docker exec $TARGET bash -c "
  mkdir -p ~/.cache && cd ~/.cache
  tar xzf /tmp/chromium.tar.gz
"

# 5. 安装系统依赖（需要 root）
docker exec -u root $TARGET bash -c "
  apt-get update -qq
  apt-get install -y libnspr4 libnss3 libatk1.0-0 libatk-bridge2.0-0 \
    libdbus-1-3 libcups2 libxkbcommon0 libatspi2.0-0 libxcomposite1 \
    libxdamage1 libxfixes3 libxrandr2 libgbm1 libasound2
"
```

**验证**:
```bash
docker exec $TARGET python -c "from playwright.sync_api import sync_playwright; print('OK')"
```

---

### 技巧 3：多实例端口规划

| 实例 | Gateway | 浏览器控制 | 用途 |
|------|---------|-----------|------|
| main | 18789 | 18791 | 主实例 |
| trump | 9999 | 9998 | Telegram/Discord Bot |
| anna | 10000 | 9996 | 第二 Bot |

**端口映射**:
```bash
-p 10000:18789  # Gateway（对外访问）
-p 9996:18791   # 浏览器控制（Playwright）
```

---

## 📝 完整初始化脚本

复制以下脚本一键完成所有配置：

```bash
#!/bin/bash

echo "🚀 创建并启动容器..."
docker run -d \
  --name stock \
  -p 9999:18789 \
  -v ~/.openclaw-stock:/home/node/.openclaw \
  -e OPENCLAW_GATEWAY_PORT=18789 \
  ghcr.io/openclaw/openclaw:2026.4.2

echo "⏳ 等待容器初始化..."
sleep 5

echo "⚙️ 配置绑定模式..."
docker exec stock node dist/index.js config set gateway.bind "lan"

echo "⚙️ 配置允许的源..."
docker exec stock node dist/index.js config set gateway.controlUi.allowedOrigins '["http://localhost:9999","http://127.0.0.1:9999"]'

echo "⚙️ 设置 Token..."
docker exec stock node dist/index.js config set gateway.auth.token "hope"

echo "⚙️ 配置百炼 Provider..."
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
    }
  ]
}'

echo "⚙️ 设置默认模型..."
docker exec stock node dist/index.js config set agents.defaults.model "bailian/qwen3.5-plus"

echo "🔄 重启容器..."
docker restart stock

echo "⏳ 等待容器重启..."
sleep 5

echo ""
echo "📊 容器状态:"
docker ps | grep stock

echo ""
echo "📁 验证挂载:"
ls -la ~/.openclaw-stock/

echo ""
echo "🔗 访问地址：http://localhost:9999/"
echo "🔑 Token: hope"
echo ""
echo "✅ 配置完成！"
```

---

## 🔗 相关资源

- **官方文档**: https://docs.openclaw.ai/install/docker
- **GitHub 镜像**: https://github.com/openclaw/openclaw/pkgs/container/openclaw
- **社区**: https://discord.com/invite/clawd
- **AI-Note 文档**: https://github.com/Linux2010/ai-note/tree/main/openclaw/best-practices

---

## 📚 变更记录

| 日期 | 版本 | 说明 |
|------|------|------|
| 2026-04-08 | 1.2 | 新增高级技巧：禁用代理、Playwright 复制法、多实例端口规划 |
| 2026-04-06 | 1.1 | 修正挂载目录为 `/home/node/.openclaw` |
| 2026-04-06 | 1.0 | 初始版本，记录完整部署流程 |

---

*本文档由 AI 生成并维护，如有问题请提交 PR 更新。*
