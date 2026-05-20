---
title: "macOS 部署 OpenClaw：TCP 连接风暴导致代理断连"
date: "2026-05-20"
author: "Leader"
tags: ["macOS", "TCP", "TIME_WAIT", "Clash Verge", "端口耗尽", "网络故障", "OpenClaw"]
related:
  - "openclaw/solutions/docker-telegram-proxy-connection-issue.md"
  - "openclaw/solutions/proxy-solutions.md"
---

# macOS TCP 连接风暴导致 OpenClaw 代理断连

> **机器**: Mac mini M4 (16GB) | **日期**: 2026-05-20  
> **影响**: OpenClaw Telegram + LLM 全部不可用，持续约 2 小时

## 问题现象

Mac mini 部署的 OpenClaw 代理突然无法回复 Telegram 消息：

```
telegram message failed: Network request for 'sendMessage' failed!
LLM request failed: network connection error.
telegram getUpdates: stall detected
```

## 排查结果

### 根因：系统 TCP 连接表被撑满

```bash
$ netstat -an | awk '/^tcp/ {print $6}' | sort | uniq -c | sort -rn
  16184 TIME_WAIT
    822 FIN_WAIT_1
     18 ESTABLISHED
```

macOS 临时端口范围仅 49152-65535（**16384 个**），TIME_WAIT 几乎占满。新连接无法分配端口，报错 `Can't assign requested address`。

### 连接来源

**TIME_WAIT 16184（Clash Verge mihomo 进程产生）**：

| 目标 | 数量 | 说明 |
|------|------|------|
| 127.0.0.1:7897（代理口） | 8392 | Docker 卡死连接断开后堆积 |
| DNS over TLS（223.5.5.5:853） | ~2400 | mihomo 频繁 DNS 查询 |
| 机场节点健康检查 | ~2500 | url-test 30 分钟轮询 |
| GeoIP/规则下载 | ~2900 | Cloudflare CDN |

**FIN_WAIT_1 822（死节点卡死）**：

| 目标 | 数量 | 说明 |
|------|------|------|
| 42.200.172.3:6001（台湾节点） | 495 | **节点已死，FIN 无响应** |
| 36.230.38.108:6001（台湾） | 76 | 不可达 |
| 36.230.30.217:6001（台湾） | 51 | 不可达 |
| 209.9.201.34:6001（美国） | 45 | 不可达 |
| Apple Push 通知 | ~70 | 系统服务 |

### 关键发现：macOS TCP 栈 Bug

- `net.inet.tcp.msl` = 15000ms，TIME_WAIT 应 30 秒超时
- **实测 80+ 秒完全不释放**，数量永远不变
- 杀掉所有相关进程（mihomo、Docker）也无效
- **只能重启机器**才能清理

这是 macOS 特有的 TCP 状态机冻结 bug，Linux 上不存在。

## 解决方案

### 紧急修复

```bash
# 1. 重启机器（唯一有效方法）
sudo shutdown -r now

# 2. 重启后手动启动代理（比等 GUI 快）
mkdir -p /tmp/verge
nohup /Applications/Clash\ Verge.app/Contents/MacOS/verge-mihomo \
  -d "~/Library/Application Support/io.github.clash-verge-rev.clash-verge-rev" \
  -f "~/Library/Application Support/io.github.clash-verge-rev.clash-verge-rev/clash-verge.yaml" \
  -ext-ctl-unix /tmp/verge/verge-mihomo.sock > /tmp/mihomo.log 2>&1 &

# 3. 切换到可用节点
curl -s -X PUT -H "Content-Type: application/json" \
  --unix-socket /tmp/verge/verge-mihomo.sock \
  "http://localhost/proxies/三毛机场" \
  -d '{"name":"新加坡1|高速下载|移动优化"}'

# 4. 启动 OpenClaw
openclaw gateway restart
```

### 预防措施

1. **减少 url-test 频率**：interval 从 1800s 改为 3600s+
2. **移除死节点**：定期测试，剔除不可用节点
3. **定期重启**：长期运行的 Mac 建议每周重启
4. **监控 TCP 连接数**：

```bash
# 添加到 cron，超过阈值告警
count=$(netstat -an 2>/dev/null | grep -c "^tcp")
if [ "$count" -gt 5000 ]; then
  echo "⚠️ TCP connections: $count"
fi
```

## 诊断命令速查

```bash
# 检查 TCP 连接数
netstat -an | awk '/^tcp/ {print $6}' | sort | uniq -c | sort -rn

# 查看 TIME_WAIT 目标分布
netstat -an | grep TIME_WAIT | awk '{print $5}' | awk -F: '{print $1}' | sort | uniq -c | sort -rn

# 查看 FIN_WAIT_1 详情
netstat -an | grep FIN_WAIT_1 | head -20

# 检查端口监听
lsof -i :7897 -sTCP:LISTEN

# 测试代理连通性
curl -s --max-time 5 --proxy http://127.0.0.1:7897 https://www.google.com -o /dev/null -w "%{http_code}"
```
