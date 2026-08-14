# OpenClaw 核心架构与消息处理循环

> 深入理解 OpenClaw 的设计理念、架构分层和 Agent 与基座模型的交互机制

**文档类型**: 架构设计
**适用场景**: 理解 OpenClaw 内部实现、排查消息处理问题、开发插件
**最后更新**: 2026-04-22

---

## 一、设计理念

### 1. 核心愿景

> **OpenClaw is the AI that actually does things.**
> It runs on your devices, in your channels, with your rules.

**三大核心原则**：

| 原则 | 描述 |
|------|------|
| **个人化** | 在你的设备上运行，尊重隐私 |
| **通道化** | 在你已有的消息平台上工作（Telegram、Discord、WhatsApp 等） |
| **可控性** | 按你的规则行事，安全默认 |

### 2. 技术选择

| 选择 | 原因 |
|------|------|
| **TypeScript** | OpenClaw 是编排系统，TypeScript 保持"可 hack"：易读、易修改、易扩展 |
| **插件优先** | Core 保持精简，可选能力通过插件实现 |
| **事件驱动** | WebSocket 监听消息，异步处理，高并发 |

---

## 二、架构总览

### 五层架构

| 层级 | 文件 | 核心功能 |
|------|------|----------|
| **L1 服务器启动** | `src/gateway/server.impl.ts` | 创建 HTTP/WebSocket 服务器，启动运行时服务 |
| **L2 WebSocket 处理** | `src/gateway/server/ws-connection/message-handler.ts` | 消息循环入口，JSON frame 解析 |
| **L3 请求路由** | `src/gateway/server-methods.ts` | 根据 method 名分发到 handlers |
| **L4 Handlers** | `src/gateway/server-methods/*.ts` | 30+ 业务 handler (chat, agent, cron...) |
| **L5 Agent 执行** | `src/agents/pi-embedded-runner/run/attempt.ts` | 模型推理，Tool Call 循环 |

### 架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         OpenClaw Architecture                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  用户消息                                                            │
│      ↓                                                              │
│  WebSocket Server (socket.on("message"))                            │
│      ↓                                                              │
│  JSON.parse → {type, method, id, params}                            │
│      ↓                                                              │
│  handleGatewayRequest()                                             │
│      ↓                                                              │
│  handlers[method] → chatHandlers.agent.run                          │
│      ↓                                                              │
│  SessionManager → 加载历史对话                                        │
│      ↓                                                              │
│  createAgentSession() → 创建 Agent                                   │
│      ↓                                                              │
│  ┌─────────────────────────────────────────────┐                   │
│  │         PI Agent Core Loop                  │                   │
│  ├─────────────────────────────────────────────┤                   │
│  │  while (!stop) {                            │                   │
│  │    response = await callModel();            │                   │
│  │    for block in response:                   │                   │
│  │      if (text): emit → 用户                 │                   │
│  │      if (tool_call):                        │                   │
│  │        result = executeTool();              │                   │
│  │        messages.push(tool_result);          │                   │
│  │        continue; // 重新调用                 │                   │
│  │    if (stopReason === "stop"): break;       │                   │
│  │  }                                          │                   │
│  └─────────────────────────────────────────────┘                   │
│      ↓                                                              │
│  WebSocket → 用户收到回复                                            │
│      ↓                                                              │
│  SessionManager 保存 transcript                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 三、WebSocket 消息循环

### 消息处理流程

```typescript
// message-handler.ts 核心实现
socket.on("message", async (data) => {
  // 1. 解析 JSON frame
  const parsed = JSON.parse(text);
  
  // 2. 验证 frame 格式
  // { type:"req", method:"xxx", id:"xxx", params:{...} }
  
  // 3. 如果未认证 → 处理 connect handshake
  if (!client) {
    await handleConnectHandshake(parsed);
    return;
  }
  
  // 4. 已认证 → 分发请求
  handleGatewayRequest({
    req: parsed,
    client,
    respond: (ok, result, error) => send({type:"res", id, ok, result, error})
  });
});
```

### JSON Frame 格式

**请求**:
```json
{"type": "req", "method": "chat", "id": "uuid", "params": {"body": "消息"}}
```

**响应**:
```json
{"type": "res", "id": "uuid", "ok": true, "result": {...}}
```

---

## 四、请求路由层

### handleGatewayRequest 实现

```typescript
export async function handleGatewayRequest(opts) {
  const { req, respond, client } = opts;
  
  // 1. 权限检查
  const authError = authorizeGatewayMethod(req.method, client);
  
  // 2. 获取 handler
  const handler = coreGatewayHandlers[req.method];
  
  // 3. 执行 handler
  await handler({req, params, client, respond, context});
}
```

### 注册的 Handlers

- `connectHandlers` - WebSocket 连接
- `chatHandlers` - 聊天消息
- `agentHandlers` - Agent 运行
- `configHandlers` - 配置管理
- `cronHandlers` - 定时任务
- `sessionsHandlers` - Session 管理
- `deviceHandlers` - 设备配对
- `channelsHandlers` - 频道管理
- `toolsCatalogHandlers` - 工具目录
- ... 30+ handlers

---

## 五、Agent 执行层

### Agent 与基座模型交互流程

1. `prepareAgentCommandExecution()` → 准备执行上下文
2. `SessionManager.open(sessionFile)` → 加载历史对话
3. `createAgentSession()` → 创建 Agent Session
4. `buildEmbeddedSystemPrompt()` → 构建系统提示词
5. `activeSession.prompt(userInput)` → 提交用户输入
6. `agent.streamFn(model, messages)` → 流式调用模型 API
7. **PI Agent Core 内部循环** → 处理 tool calls
8. Tool Result 返回 → 自动重新调用模型
9. Stop Reason → 结束 loop
10. `persistTurnTranscript()` → 保存历史

---

## 六、Tool Call 循环机制

### PI Agent Core 内部循环

```typescript
async function runAgentLoop() {
  while (true) {
    const response = await streamFn(model, {messages, tools});
    
    for (const block of response.content) {
      if (block.type === "text") {
        emitAgentEvent({type: "assistant_text", text: block.text});
      }
      if (block.type === "tool_use") {
        const result = await executeTool({name, input});
        messages.push({role: "assistant", content: [block]});
        messages.push({role: "user", content: [{type: "tool_result", content: result}]});
        continue; // 重新调用模型
      }
    }
    
    if (response.stopReason === "stop") break;
  }
}
```

### 内置工具列表

- `read` - 读取文件
- `write` - 写入文件
- `edit` - 编辑文件
- `exec` - 执行命令
- `browser` - 浏览器操作
- `web_search` - 网络搜索
- `web_fetch` - 网页抓取
- `memory_search` - 记忆搜索
- `message` - 发送消息
- `tts` - 语音合成
- ... 30+ 工具

---

## 七、运行时服务

| 服务 | 功能 |
|------|------|
| **HeartbeatRunner** | 每 30 分钟检查活跃 session，执行监控任务 |
| **ChannelHealthMonitor** | 每 5 分钟检查频道健康，自动重启崩溃进程 |
| **CronService** | 执行定时任务，isolated agent turns |
| **ModelPricingRefresh** | 定期刷新模型定价数据 |
| **Compaction** | 提前压缩对话历史避免 context overflow |

---

## 八、关键设计原则

| 原则 | 实现方式 |
|------|----------|
| **WebSocket 事件驱动** | `socket.on("message")` 监听消息 |
| **Handler 路由** | 根据 method 名分发请求 |
| **异步执行** | Agent 运行使用 async/await 流式处理 |
| **Tool 自动循环** | PI Core 自动处理 tool calls |
| **Session 持久化** | SessionManager 保存 transcript |
| **Compaction 保护** | 提前压缩避免 context overflow |
| **Plugin 优先** | Core 精简，扩展通过插件 |

---

## 参考资料

- **源码位置**:
  - Gateway: `src/gateway/server.impl.ts`
  - WebSocket: `src/gateway/server/ws-connection/message-handler.ts`
  - Handlers: `src/gateway/server-methods.ts`
  - Agent: `src/agents/pi-embedded-runner/run/attempt.ts`
  - Tools: `src/agents/pi-tools.ts`

- **官方文档**: https://docs.openclaw.ai
- **GitHub**: https://github.com/openclaw/openclaw

---

*最后更新: 2026-04-22*