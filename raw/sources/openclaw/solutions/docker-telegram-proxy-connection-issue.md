# Docker 容器内 Telegram Bot 连接问题排查

**创建时间：** 2026-03-29  
**问题类型：** 网络代理 / Docker 容器  
**影响范围：** Docker 部署的 OpenClaw + Telegram 集成

---

## 🔍 问题现象

### 症状
- Telegram Bot Web 界面可正常访问 (`http://127.0.0.1:28789/`)
- Bot API 从主机测试正常 (`curl -x http://127.0.0.1:7897 https://api.telegram.org/...`)
- **但容器内的 OpenClaw 无法接收 Telegram 消息**
- 用户在 Telegram 发送消息后，Bot 不回复

### 用户反馈
> "我在 Telegram 给他发信息，他不回复我呢。是通道有问题吗？"

---

## 📊 诊断过程

### 1. 检查 Bot 在线状态 ✅
```bash
curl -s -x http://127.0.0.1:7897 \
  "https://api.telegram.org/bot8769374284:AAG2-sUBIa7TrPn3G82oZ9Nx1e1QpMavN-c/getMe"

# 结果：Bot 在线，配置正常
{"ok":true,"result":{"id":8769374284,"is_bot":true,"username":"one_hope_trump_bot",...}}
```

### 2. 检查消息发送能力 ✅
```bash
curl -s -x http://127.0.0.1:7897 \
  "https://api.telegram.org/bot8769374284:AAG2-sUBIa7TrPn3G82oZ9Nx1e1QpMavN-c/sendMessage" \
  -d "chat_id=5520269161" -d "text=测试消息"

# 结果：消息发送成功 (message_id: 44)
{"ok":true,"result":{"message_id":44,...}}
```

### 3. 检查容器内代理连接 ❌
```bash
# 主机测试 - 成功
curl -x http://127.0.0.1:7897 https://api.telegram.org/  # ✅ HTTP 200

# 容器内测试 - 失败
curl -x http://host.docker.internal:7897 https://api.telegram.org/  # ❌ 错误 5 (无法解析)
```

### 4. 检查 OpenClaw 配置
```json
// ~/.openclaw-trump/openclaw.json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "8769374284:AAG2-sUBIa7TrPn3G82oZ9Nx1e1QpMavN-c",
      "allowFrom": ["5520269161"],
      "proxy": "http://host.docker.internal:7897"  // ← 问题可能在这里
    }
  }
}
```

---

## 🎯 根因分析

### 问题原因

**Docker 容器内的 `host.docker.internal` DNS 解析偶尔失效**

| 测试场景 | 结果 | 说明 |
|---------|------|------|
| 主机 → Telegram (通过代理) | ✅ 正常 | 代理服务本身正常 |
| 容器内 → 主机代理 | ❌ 失败 | `host.docker.internal` 无法解析 |
| 重启容器后 | ✅ 恢复 | DNS 缓存刷新，临时恢复 |

### 为什么重启后就好了？

1. **DNS 缓存刷新** - Docker 重启后重新解析 `host.docker.internal`
2. **网络连接重置** - 容器与主机的网络桥接重新建立
3. **代理服务状态更新** - 代理可能也同时重启，重新绑定接口

---

## ✅ 解决方案

### 方案一：重启容器（临时解决）

```bash
# 重启 OpenClaw 容器
docker restart openclaw-trump

# 验证状态
docker ps | grep openclaw-trump
curl http://127.0.0.1:28789/  # 应返回 HTML
```

**优点：** 快速恢复服务  
**缺点：** 问题可能复发

---

### 方案二：使用主机固定 IP（推荐）

1. **获取主机局域网 IP**
```bash
node -e "const os = require('os'); const nets = os.networkInterfaces(); 
for(const name of Object.keys(nets)){
  for(const net of nets[name]){
    if(net.family==='IPv4'&&!net.internal){
      console.log(name+': '+net.address);
    }
  }
}"

# 输出：en0: 192.168.31.25
```

2. **修改 OpenClaw 配置**
```json
// ~/.openclaw-trump/openclaw.json
{
  "channels": {
    "telegram": {
      "proxy": "http://192.168.31.25:7897"  // 使用固定 IP
    }
  }
}
```

3. **重启容器**
```bash
docker restart openclaw-trump
```

**优点：** 稳定可靠，避免 DNS 解析问题  
**缺点：** 主机 IP 变更时需要更新配置

---

### 方案三：配置 Docker 网络别名

```bash
# 启动时添加额外 Host 映射
docker run -d --name openclaw-trump \
  --add-host=host.docker.internal:host-gateway \
  -p 28789:18789 -p 28791:18791 \
  -v ~/.openclaw-trump:/home/node/.openclaw \
  ghcr.io/openclaw/openclaw:latest
```

**优点：** 标准化方案，Docker 官方支持  
**缺点：** 需要重新创建容器

---

## 🛠️ 诊断命令清单

### 检查 Bot 状态
```bash
# Bot 在线状态
curl -x http://127.0.0.1:7897 \
  "https://api.telegram.org/bot<TOKEN>/getMe"

# 获取最新消息更新
curl -x http://127.0.0.1:7897 \
  "https://api.telegram.org/bot<TOKEN>/getUpdates?offset=0&limit=5"

# 发送测试消息
curl -x http://127.0.0.1:7897 \
  "https://api.telegram.org/bot<TOKEN>/sendMessage" \
  -d "chat_id=<USER_ID>" -d "text=测试消息"
```

### 检查容器状态
```bash
# 容器运行状态
docker ps | grep openclaw-trump

# 容器日志
docker logs openclaw-trump --tail 50

# 容器内网络测试
docker exec openclaw-trump curl -x http://host.docker.internal:7897 \
  https://api.telegram.org/
```

### 检查配置文件
```bash
# Telegram 配置
cat ~/.openclaw-trump/openclaw.json | grep -A 10 '"telegram"'

# Update Offset（最后处理的消息 ID）
cat ~/.openclaw-trump/telegram/update-offset-default.json
```

---

## 📋 预防措施

### 1. 监控容器健康状态
```bash
# 定期检查容器健康
docker inspect openclaw-trump --format='{{.State.Health.Status}}'

# 设置自动重启
docker update --restart=always openclaw-trump
```

### 2. 配置日志持久化
```bash
# 确保日志目录挂载
docker run -d --name openclaw-trump \
  -v ~/.openclaw-trump/logs:/home/node/.openclaw/logs \
  ...
```

### 3. 设置健康检查
```bash
# 添加健康检查脚本
docker inspect openclaw-trump --format='{{.Config.Healthcheck}}'
```

---

## 🔗 相关文档

- [OpenClaw Docker 部署指南](https://docs.openclaw.ai/deployment/docker)
- [Telegram Bot API](https://core.telegram.org/bots/api)
- [Docker 网络配置](https://docs.docker.com/network/)
- [proxy-solutions.md](./proxy-solutions.md) - 代理配置综合方案

---

## 📝 更新记录

| 日期 | 内容 | 作者 |
|------|------|------|
| 2026-03-29 | 初始版本 - 记录 Docker 容器内代理连接问题排查 | Andy |

---

**标签：** #OpenClaw #Telegram #Docker #代理 #故障排查  
**适用版本：** OpenClaw v2026+, Docker Desktop for Mac
