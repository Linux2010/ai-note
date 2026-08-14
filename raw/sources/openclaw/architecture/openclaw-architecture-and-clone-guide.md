# OpenClaw 架构设计与复刻指南

> **版本**: 1.0  
> **最后更新**: 2026-05-11  
> **目标**: 为复刻一个类似 OpenClaw 的 AI Agent 系统提供完整的架构设计和实现指导

---

## 目录

1. [设计理念与愿景](#一设计理念与愿景)
2. [五层架构详解](#二五层架构详解)
3. [WebSocket 消息循环](#三websocket-消息循环)
4. [插件系统设计](#四插件系统设计)
5. [工具系统架构](#五工具系统架构)
6. [Session 管理机制](#六session-管理机制)
7. [认证与权限系统](#七认证与权限系统)
8. [配置系统深度解析](#八配置系统深度解析)
9. [Memory 系统整合](#九memory-系统整合)
10. [多 Agent 架构](#十多-agent-架构)
11. [部署方案](#十一部署方案)
12. [复刻实施指南](#十二复刻实施指南)

---

## 一、设计理念与愿景

### 1.1 核心愿景

> **OpenClaw is the AI that actually does things.**
> It runs on your devices, in your channels, with your rules.

OpenClaw 的核心定位是：一个**能真正执行任务的 AI Agent**，而非仅仅是一个对话机器人。

### 1.2 三大核心原则

| 原则 | 描述 | 复刻要点 |
|------|------|----------|
| **个人化 (Personal)** | 在用户设备上运行，尊重隐私 | 本地部署优先，数据不外传 |
| **通道化 (Channel)** | 在用户已有的消息平台上工作 | 支持 Telegram/Discord/Slack 等 |
| **可控性 (Controllable)** | 按用户规则行事，安全默认 | 权限最小化，用户确认机制 |

### 1.3 技术选择哲学

| 选择 | 原因 | 复刻替代方案 |
|------|------|--------------|
| **TypeScript** | OpenClaw 是编排系统，TS 保持"可 hack"：易读、易修改、易扩展 | Python/Go/Rust |
| **插件优先** | Core 保持精简，可选能力通过插件实现 | 模块化设计 |
| **事件驱动** | WebSocket 监听消息，异步处理，高并发 | 消息队列架构 |

### 1.4 与其他 Agent 框架对比

| 框架 | 定位 | 架构特点 | 适用场景 |
|------|------|----------|----------|
| **OpenClaw** | 个人 AI 助手 | TypeScript + WebSocket + 插件 | 本地运行、多通道 |
| **LangGraph** | Agent 编排框架 | Python + 图状态机 | 复杂流程编排 |
| **AutoGPT** | 自主 Agent | Python + 任务循环 | 自动化任务 |
| **CrewAI** | 多 Agent 协作 | Python + 角色 | 团队协作场景 |

---

## 二、五层架构详解

### 2.1 架构总览

OpenClaw 采用**五层架构设计**，每层职责清晰，便于复刻时逐层实现。

```
┌─────────────────────────────────────────────────────────────────────┐
│                         OpenClaw Architecture                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  用户消息                                                            │
│      ↓                                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ L1 服务器启动层 (Gateway Server)                              │   │
│  │ - 创建 HTTP/WebSocket 服务器                                   │   │
│  │ - 启动运行时服务 (Heartbeat, Cron, HealthMonitor)             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│      ↓                                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ L2 WebSocket 处理层 (Message Handler)                         │   │
│  │ - JSON frame 解析                                             │   │
│  │ - 认证 handshake                                              │   │
│  │ - 消息循环入口                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│      ↓                                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ L3 请求路由层 (Gateway Request Handler)                       │   │
│  │ - method 分发                                                 │   │
│  │ - 权限检查                                                    │   │
│  │ - handler 注册                                                │   │
│  └─────────────────────────────────────────────────────────────┘   │
│      ↓                                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ L4 Handlers 层 (Business Handlers)                           │   │
│  │ - chat, agent, cron, config, device, channels...              │   │
│  │ - 30+ 业务 handler                                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│      ↓                                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ L5 Agent 执行层 (PI Agent Core)                              │   │
│  │ - 模型推理                                                    │   │
│  │ - Tool Call 循环                                              │   │
│  │ - Session 管理                                                │   │
│  └─────────────────────────────────────────────────────────────┘   │
│      ↓                                                              │
│  WebSocket → 用户收到回复                                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 各层详解

#### L1 服务器启动层

**职责**: 创建服务器，启动运行时服务

**核心文件**: `src/gateway/server.impl.ts`

**主要功能**:
```typescript
// 服务器启动核心逻辑
async function startGateway() {
  // 1. 创建 HTTP Server
  const httpServer = createServer(app);
  
  // 2. 创建 WebSocket Server
  const wsServer = new WebSocketServer({ server: httpServer });
  
  // 3. 启动运行时服务
  await HeartbeatRunner.start();      // 心跳检查
  await ChannelHealthMonitor.start(); // 频道健康监控
  await CronService.start();          // 定时任务
  await ModelPricingRefresh.start();  // 模型定价刷新
}
```

**运行时服务列表**:

| 服务 | 功能 | 启动时机 |
|------|------|----------|
| **HeartbeatRunner** | 每 30 分钟检查活跃 session，执行监控任务 | Gateway 启动 |
| **ChannelHealthMonitor** | 每 5 分钟检查频道健康，自动重启崩溃进程 | Gateway 启动 |
| **CronService** | 执行定时任务，isolated agent turns | Gateway 启动 |
| **ModelPricingRefresh** | 定期刷新模型定价数据 | Gateway 启动 |

**复刻要点**:
```python
# Python 版本复刻示例
class GatewayServer:
    def __init__(self, port: int = 18789):
        self.port = port
        self.http_server = None
        self.ws_server = None
        self.runtime_services = []
    
    async def start(self):
        # 1. 创建 HTTP Server
        self.http_server = await aiohttp.web.Application()
        
        # 2. 创建 WebSocket Server
        self.ws_server = WebSocketServer(self.http_server)
        
        # 3. 启动运行时服务
        self.runtime_services = [
            HeartbeatRunner(),
            ChannelHealthMonitor(),
            CronService(),
        ]
        for service in self.runtime_services:
            await service.start()
```

---

#### L2 WebSocket 处理层

**职责**: 消息解析、认证 handshake、消息循环入口

**核心文件**: `src/gateway/server/ws-connection/message-handler.ts`

**消息处理流程**:
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

**JSON Frame 格式**:

```json
// 请求格式
{
  "type": "req",
  "method": "chat",
  "id": "uuid-xxx",
  "params": {
    "body": "用户消息内容"
  }
}

// 响应格式
{
  "type": "res",
  "id": "uuid-xxx",
  "ok": true,
  "result": { ... }
}

// 错误响应
{
  "type": "res",
  "id": "uuid-xxx",
  "ok": false,
  "error": "错误信息"
}

// 事件流（Agent 输出）
{
  "type": "event",
  "id": "uuid-xxx",
  "event": {
    "type": "assistant_text",
    "text": "AI 输出的文本"
  }
}
```

**复刻要点**:
```python
# Python 版本 WebSocket Handler
class MessageHandler:
    async def on_message(self, websocket, data: str):
        try:
            frame = json.loads(data)
            
            # 验证 frame 格式
            if not self.validate_frame(frame):
                await self.send_error(websocket, "Invalid frame format")
                return
            
            # 检查认证状态
            client = self.get_client(websocket)
            if not client:
                await self.handle_handshake(websocket, frame)
                return
            
            # 分发请求
            await self.dispatch_request(websocket, frame, client)
            
        except json.JSONDecodeError:
            await self.send_error(websocket, "Invalid JSON")
```

---

#### L3 请求路由层

**职责**: method 分发、权限检查、handler 注册

**核心文件**: `src/gateway/server-methods.ts`

**路由实现**:
```typescript
export async function handleGatewayRequest(opts) {
  const { req, respond, client } = opts;
  
  // 1. 权限检查
  const authError = authorizeGatewayMethod(req.method, client);
  if (authError) {
    respond(false, null, authError);
    return;
  }
  
  // 2. 获取 handler
  const handler = coreGatewayHandlers[req.method];
  if (!handler) {
    respond(false, null, `Unknown method: ${req.method}`);
    return;
  }
  
  // 3. 执行 handler
  await handler({req, params, client, respond, context});
}
```

**Handler 注册表**:

```typescript
const coreGatewayHandlers = {
  // 连接相关
  'connect': connectHandlers.connect,
  'pair': connectHandlers.pair,
  
  // 聊天相关
  'chat': chatHandlers.chat,
  'chat.stream': chatHandlers.stream,
  
  // Agent 相关
  'agent.run': agentHandlers.run,
  'agent.spawn': agentHandlers.spawn,
  'agent.stop': agentHandlers.stop,
  
  // Session 相关
  'sessions.list': sessionsHandlers.list,
  'sessions.get': sessionsHandlers.get,
  'sessions.delete': sessionsHandlers.delete,
  
  // 配置相关
  'config.get': configHandlers.get,
  'config.set': configHandlers.set,
  
  // 定时任务
  'cron.list': cronHandlers.list,
  'cron.create': cronHandlers.create,
  'cron.delete': cronHandlers.delete,
  
  // 设备管理
  'devices.list': deviceHandlers.list,
  'devices.approve': deviceHandlers.approve,
  
  // 频道管理
  'channels.list': channelsHandlers.list,
  'channels.configure': channelsHandlers.configure,
  
  // 工具目录
  'tools.list': toolsCatalogHandlers.list,
  
  // ... 30+ handlers
};
```

**复刻要点**:
```python
# Python 版本路由器
class GatewayRouter:
    handlers: Dict[str, Callable] = {}
    
    def register_handler(self, method: str, handler: Callable):
        self.handlers[method] = handler
    
    async def dispatch(self, method: str, params: dict, client: Client):
        # 1. 权限检查
        if not self.check_permission(method, client):
            return {"ok": False, "error": "Permission denied"}
        
        # 2. 获取 handler
        handler = self.handlers.get(method)
        if not handler:
            return {"ok": False, "error": f"Unknown method: {method}"}
        
        # 3. 执行 handler
        result = await handler(params, client)
        return {"ok": True, "result": result}
```

---

#### L4 Handlers 层

**职责**: 业务逻辑处理

**核心 Handler 分类**:

| 类别 | Handlers | 功能 |
|------|----------|------|
| **连接** | connect, pair | WebSocket 连接、设备配对 |
| **聊天** | chat, chat.stream | 消息处理、流式输出 |
| **Agent** | run, spawn, stop | Agent 执行、子 Agent、停止 |
| **Session** | list, get, delete | Session 管理 |
| **配置** | get, set | 配置读写 |
| **定时** | list, create, delete, run | Cron 任务管理 |
| **设备** | list, approve, reject | 设备配对管理 |
| **频道** | list, configure | Telegram/Discord 配置 |

**chat Handler 核心逻辑**:
```typescript
async function handleChat({params, client, respond, context}) {
  const { agentId, body, sessionId } = params;
  
  // 1. 获取或创建 Session
  const session = await SessionManager.getOrCreate(sessionId, agentId);
  
  // 2. 准备 Agent 执行上下文
  const agentContext = await prepareAgentCommandExecution({
    agentId,
    sessionId,
    userInput: body
  });
  
  // 3. 执行 Agent
  await runAgent(agentContext, {
    onEvent: (event) => {
      // 流式发送事件
      respond({type: "event", event});
    },
    onComplete: (result) => {
      respond({type: "res", ok: true, result});
    }
  });
}
```

---

#### L5 Agent 执行层

**职责**: 模型推理、Tool Call 循环、Session 管理

**核心文件**: `src/agents/pi-embedded-runner/run/attempt.ts`

**Agent 与基座模型交互流程**:

```
1. prepareAgentCommandExecution() → 准备执行上下文
2. SessionManager.open(sessionFile) → 加载历史对话
3. createAgentSession() → 创建 Agent Session
4. buildEmbeddedSystemPrompt() → 构建系统提示词
5. activeSession.prompt(userInput) → 提交用户输入
6. agent.streamFn(model, messages) → 流式调用模型 API
7. PI Agent Core 内部循环 → 处理 tool calls
8. Tool Result 返回 → 自动重新调用模型
9. Stop Reason → 结束 loop
10. persistTurnTranscript() → 保存历史
```

---

## 三、WebSocket 消息循环

### 3.1 消息流向

```
用户 → WebSocket → MessageHandler → GatewayRouter → Handler → Agent Core → Tool → 结果 → WebSocket → 用户
```

### 3.2 消息类型

| 类型 | 方向 | 用途 |
|------|------|------|
| `req` | Client → Server | 请求消息 |
| `res` | Server → Client | 响应消息 |
| `event` | Server → Client | 事件流（Agent 输出） |

### 3.3 连接状态管理

```typescript
enum ClientState {
  CONNECTING,   // 正在连接
  AUTHENTICATING, // 正在认证
  PAIRING,      // 等待配对
  ACTIVE,       // 已激活
  DISCONNECTED  // 已断开
}

class ClientConnection {
  websocket: WebSocket;
  state: ClientState;
  deviceId: string;
  agentBindings: AgentBinding[];
}
```

---

## 四、插件系统设计

### 4.1 插件架构概览

OpenClaw 采用 **Slot + Entry** 双层插件架构：

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Plugin Architecture                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Slots (插槽层)                                                │   │
│  │ - 定义可替换的功能模块                                         │   │
│  │ - 如: memory, browser, search                                │   │
│  └─────────────────────────────────────────────────────────────┘   │
│      ↓ (绑定)                                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Entries (插件实例层)                                          │   │
│  │ - 具体插件的配置和实现                                         │   │
│  │ - 如: memory-lancedb, playwright, brave                      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 Slot 机制

**Slot 定义**: 可替换的功能模块"插槽"

**内置 Slots**:

| Slot | 用途 | 默认实现 |
|------|------|----------|
| `memory` | 记忆系统 | `memory-core` |
| `browser` | 浏览器自动化 | `playwright` |
| `search` | 网络搜索 | `brave` |

**Slot 配置**:
```json
{
  "plugins": {
    "slots": {
      "memory": "memory-lancedb-pro",
      "browser": "playwright",
      "search": "brave"
    }
  }
}
```

### 4.3 Entry 机制

**Entry 定义**: 具体插件的配置实例

**Entry 配置**:
```json
{
  "plugins": {
    "entries": {
      "memory-lancedb-pro": {
        "enabled": true,
        "config": {
          "embedding": {
            "provider": "ollama",
            "model": "nomic-embed-text"
          },
          "autoCapture": true,
          "autoRecall": true
        }
      },
      "playwright": {
        "enabled": true,
        "config": {
          "headless": true,
          "browser": "chromium"
        }
      },
      "brave": {
        "enabled": true,
        "config": {
          "apiKey": "${BRAVE_API_KEY}"
        }
      }
    }
  }
}
```

### 4.4 插件生命周期

```
┌─────────────┐
│  加载阶段    │ - 从配置读取 slots 和 entries
├─────────────┤
│  注册阶段    │ - 注册工具、事件监听器
├─────────────┤
│  初始化阶段  │ - 调用 plugin.init(config)
├─────────────┤
│  运行阶段    │ - 处理工具调用、事件
├─────────────┤
│  清理阶段    │ - plugin.cleanup()
└─────────────┘
```

### 4.5 插件接口规范

**复刻时需实现的插件接口**:

```python
# Python 版本插件接口
from abc import ABC, abstractmethod
from typing import Dict, Any, List, Callable

class PluginInterface(ABC):
    """插件基础接口"""
    
    @property
    @abstractmethod
    def name(self) -> str:
        """插件名称"""
        pass
    
    @property
    @abstractmethod
    def slot(self) -> str:
        """绑定的 slot 名称"""
        pass
    
    @property
    @abstractmethod
    def tools(self) -> List[str]:
        """提供的工具列表"""
        pass
    
    @abstractmethod
    async def init(self, config: Dict[str, Any]):
        """初始化插件"""
        pass
    
    @abstractmethod
    async def execute_tool(self, tool_name: str, params: Dict[str, Any]) -> Any:
        """执行工具"""
        pass
    
    @abstractmethod
    async def cleanup(self):
        """清理资源"""
        pass


class MemoryPlugin(PluginInterface):
    """Memory 插件示例"""
    
    name = "memory-lancedb"
    slot = "memory"
    tools = ["memory_store", "memory_recall", "memory_forget"]
    
    async def init(self, config: Dict[str, Any]):
        self.db = await LanceDB.connect(config.get("dbPath"))
        self.embedder = Embedder(config.get("embedding"))
    
    async def execute_tool(self, tool_name: str, params: Dict[str, Any]):
        if tool_name == "memory_store":
            return await self.store(params)
        elif tool_name == "memory_recall":
            return await self.recall(params)
        elif tool_name == "memory_forget":
            return await self.forget(params)
```

### 4.6 插件加载流程

```python
class PluginManager:
    def __init__(self):
        self.slots: Dict[str, str] = {}      # slot -> plugin_name
        self.entries: Dict[str, PluginInterface] = {}  # plugin_name -> instance
    
    async def load_plugins(self, config: Dict):
        # 1. 加载 slots 配置
        self.slots = config.get("plugins", {}).get("slots", {})
        
        # 2. 加载 entries 配置
        entries_config = config.get("plugins", {}).get("entries", {})
        
        for plugin_name, plugin_config in entries_config.items():
            if not plugin_config.get("enabled", True):
                continue
            
            # 3. 创建插件实例
            plugin_class = self.get_plugin_class(plugin_name)
            plugin = plugin_class()
            
            # 4. 初始化插件
            await plugin.init(plugin_config.get("config", {}))
            
            # 5. 注册插件
            self.entries[plugin_name] = plugin
    
    def get_tool_provider(self, tool_name: str) -> PluginInterface:
        """根据工具名查找插件"""
        for slot, plugin_name in self.slots.items():
            plugin = self.entries.get(plugin_name)
            if plugin and tool_name in plugin.tools:
                return plugin
        return None
```

---

## 五、工具系统架构

### 5.1 工具分类

OpenClaw 内置 **30+ 工具**，按功能分类：

| 类别 | 工具 | 功能 |
|------|------|------|
| **文件操作** | read, write, edit | 文件读写编辑 |
| **命令执行** | exec, bash | 执行 shell 命令 |
| **浏览器** | browser_navigate, browser_click, browser_screenshot | Playwright 操作 |
| **网络** | web_search, web_fetch | 搜索、抓取网页 |
| **记忆** | memory_search, memory_store, memory_recall | 记忆系统 |
| **消息** | message, reply | 发送消息 |
| **语音** | tts, stt | 语音合成/识别 |
| **Agent** | sessions_spawn, agent_stop | 子 Agent 调用 |
| **时间** | cron_create, cron_delete | 定时任务 |

### 5.2 工具定义规范

**工具定义 JSON Schema**:

```json
{
  "name": "read",
  "description": "读取文件内容",
  "input_schema": {
    "type": "object",
    "properties": {
      "file_path": {
        "type": "string",
        "description": "文件绝对路径"
      },
      "limit": {
        "type": "integer",
        "description": "读取行数限制",
        "default": 2000
      },
      "offset": {
        "type": "integer",
        "description": "起始行号",
        "default": 0
      }
    },
    "required": ["file_path"]
  }
}
```

### 5.3 工具执行流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Tool Execution Flow                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Agent 输出 tool_use block                                          │
│      ↓                                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 1. 解析 tool_use                                              │   │
│  │    {type: "tool_use", name: "read", input: {file_path: ...}} │   │
│  └─────────────────────────────────────────────────────────────┘   │
│      ↓                                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 2. 权限检查                                                   │   │
│  │    - 检查工具是否允许                                          │   │
│  │    - 检查参数是否合规                                          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│      ↓                                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 3. 查找工具提供者                                             │   │
│  │    - 内置工具 → Core 执行                                      │   │
│  │    - 插件工具 → PluginManager.get_tool_provider()             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│      ↓                                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 4. 执行工具                                                   │   │
│  │    result = await tool_provider.execute(name, input)          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│      ↓                                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 5. 构造 tool_result                                           │   │
│  │    {type: "tool_result", tool_use_id: "...", content: result} │   │
│  └─────────────────────────────────────────────────────────────┘   │
│      ↓                                                              │
│  返回 Agent Core → 重新调用模型                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.4 Tool Call 循环核心逻辑

```python
# Python 版本 Tool Call 循环
class AgentCore:
    async def run_loop(self, messages: List[dict], tools: List[dict]):
        while True:
            # 1. 调用模型
            response = await self.model.stream(messages, tools)
            
            # 2. 处理响应 blocks
            assistant_blocks = []
            for block in response.content:
                if block["type"] == "text":
                    # 发送文本给用户
                    await self.emit_text(block["text"])
                    assistant_blocks.append(block)
                
                elif block["type"] == "tool_use":
                    # 执行工具
                    result = await self.execute_tool(block["name"], block["input"])
                    
                    # 构造 tool_result
                    tool_result = {
                        "type": "tool_result",
                        "tool_use_id": block["id"],
                        "content": result
                    }
                    
                    assistant_blocks.append(block)
                    messages.append({"role": "assistant", "content": assistant_blocks})
                    messages.append({"role": "user", "content": [tool_result]})
                    
                    # 重新调用模型
                    continue
            
            # 3. 检查 stop reason
            if response.stop_reason == "end_turn":
                break
        
        return messages
    
    async def execute_tool(self, name: str, input: dict) -> Any:
        # 权限检查
        if not self.check_permission(name, input):
            return f"Permission denied for tool: {name}"
        
        # 查找工具提供者
        provider = self.plugin_manager.get_tool_provider(name)
        if provider:
            return await provider.execute_tool(name, input)
        
        # 内置工具
        return await self.execute_builtin_tool(name, input)
```

### 5.5 工具权限控制

**权限配置**:
```json
{
  "agents": {
    "defaults": {
      "tools": {
        "allow": ["read", "write", "exec", "web_search"],
        "deny": ["exec:rm -rf", "exec:sudo"]
      }
    },
    "list": [
      {
        "id": "stock",
        "tools": {
          "allow": ["read", "web_search", "memory_search"],
          "deny": ["exec", "write"]  // Stock Agent 不能执行命令或写文件
        }
      }
    ]
  }
}
```

---

## 六、Session 管理机制

### 6.1 Session 定义

**Session**: 一次 Agent 对话的完整上下文，包含：
- 消息历史 (messages)
- Agent 状态
- 工具调用记录
- 元数据

### 6.2 Session 文件结构

```
~/.openclaw/
└── sessions/
    └── {session-id}.json
```

**Session 文件内容**:
```json
{
  "id": "session-uuid",
  "agentId": "main",
  "createdAt": "2026-05-11T10:00:00Z",
  "updatedAt": "2026-05-11T10:30:00Z",
  "messages": [
    {"role": "user", "content": "..."},
    {"role": "assistant", "content": [...]},
    ...
  ],
  "toolCalls": [
    {"name": "read", "input": {...}, "result": ...}
  ],
  "metadata": {
    "channel": "telegram",
    "peerId": "123456"
  }
}
```

### 6.3 SessionManager 核心逻辑

```python
class SessionManager:
    def __init__(self, sessions_dir: str):
        self.sessions_dir = sessions_dir
        self.active_sessions: Dict[str, Session] = {}
    
    async def get_or_create(self, session_id: str, agent_id: str) -> Session:
        if session_id in self.active_sessions:
            return self.active_sessions[session_id]
        
        # 加载或创建 Session
        session_file = f"{self.sessions_dir}/{session_id}.json"
        if os.path.exists(session_file):
            session = await self.load_session(session_file)
        else:
            session = Session(id=session_id, agentId=agent_id)
        
        self.active_sessions[session_id] = session
        return session
    
    async def load_session(self, file_path: str) -> Session:
        with open(file_path, 'r') as f:
            data = json.load(f)
        return Session.from_dict(data)
    
    async def save_session(self, session: Session):
        session_file = f"{self.sessions_dir}/{session.id}.json"
        with open(session_file, 'w') as f:
            json.dump(session.to_dict(), f)
    
    async def close_session(self, session_id: str):
        session = self.active_sessions.get(session_id)
        if session:
            await self.save_session(session)
            del self.active_sessions[session_id]
```

### 6.4 Compaction 机制

**Compaction 目的**: 当消息历史过长时，压缩历史避免 context overflow

**Compaction 算法**:
```python
class CompactionStrategy:
    MAX_TOKENS = 100000  # 最大 token 数
    
    async def compact(self, messages: List[dict]) -> List[dict]:
        # 1. 计算 token 数
        total_tokens = self.count_tokens(messages)
        
        if total_tokens < self.MAX_TOKENS * 0.8:
            return messages  # 不需要压缩
        
        # 2. 保留最近消息
        recent_messages = messages[-20:]
        
        # 3. 压缩早期消息为摘要
        early_messages = messages[:-20]
        summary = await self.summarize(early_messages)
        
        # 4. 构造压缩后的消息
        return [
            {"role": "system", "content": f"[历史摘要] {summary}"},
            *recent_messages
        ]
    
    async def summarize(self, messages: List[dict]) -> str:
        # 使用 LLM 生成摘要
        prompt = f"请总结以下对话的关键信息:\n{self.format_messages(messages)}"
        return await self.model.generate(prompt)
```

### 6.5 Transcript 持久化

**Transcript**: Agent 执行过程的完整记录

**保存时机**:
- 每个 turn 结束后
- Session 关闭时
- Compaction 发生时

```python
async def persist_turn_transcript(session: Session, turn: Turn):
    transcript = {
        "timestamp": datetime.now().isoformat(),
        "user_input": turn.user_input,
        "assistant_response": turn.assistant_response,
        "tool_calls": turn.tool_calls,
        "tokens_used": turn.tokens_used
    }
    
    session.transcripts.append(transcript)
    await SessionManager.save_session(session)
```

---

## 七、认证与权限系统

### 7.1 Gateway 认证模式

| 模式 | 描述 | 适用场景 |
|------|------|----------|
| `none` | 无认证 | 本地开发 |
| `token` | Token 认证 | 生产环境（推荐） |
| `password` | 密码认证 | 内部使用 |
| `trusted-proxy` | 代理信任 | 企业部署 |

**Token 认证配置**:
```json
{
  "gateway": {
    "auth": {
      "mode": "token",
      "token": "your-secret-token"
    }
  }
}
```

### 7.2 设备配对机制

**配对流程**:
```
┌─────────────────────────────────────────────────────────────────────┐
│                         Pairing Flow                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 新设备连接 WebSocket                                             │
│      ↓                                                              │
│  2. 发送 connect request                                            │
│      {type: "req", method: "connect", params: {deviceId: "xxx"}}    │
│      ↓                                                              │
│  3. Gateway 检查 deviceId                                           │
│      - 已配对 → 返回 session token                                   │
│      - 未配对 → 返回 pairing_required                                │
│      ↓                                                              │
│  4. 用户在 Control UI 批准配对                                       │
│      ↓                                                              │
│  5. Gateway 生成 session token                                      │
│      ↓                                                              │
│  6. 设备使用 token 完成认证                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.3 权限最小化原则

**Agent 权限配置**:
```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "workspace": "~/.openclaw/agents/main/workspace",
        "permissions": {
          "fs": {
            "allow": ["~/.openclaw/**", "~/projects/**"],
            "deny": ["~/.ssh/**", "~/.gnupg/**"]
          },
          "tools": {
            "allow": ["*"],
            "deny": ["exec:rm -rf"]
          }
        }
      },
      {
        "id": "stock",
        "workspace": "~/.openclaw/agents/stock/workspace",
        "permissions": {
          "fs": {
            "allow": ["~/.openclaw/agents/stock/**"],
            "deny": ["*"]
          },
          "tools": {
            "allow": ["read", "web_search", "memory_search"],
            "deny": ["exec", "write"]
          }
        }
      }
    ]
  }
}
```

### 7.4 通道权限控制

**Telegram/Discord 权限**:
```json
{
  "channels": {
    "telegram": {
      "accounts": {
        "core": {
          "dmPolicy": "pairing",      // 私聊需要配对
          "groupPolicy": "allowlist", // 群组使用白名单
          "allowFrom": [123456789],   // 允许的用户 ID
          "allowBots": false          // 不允许其他 Bot
        }
      }
    }
  }
}
```

---

## 八、配置系统深度解析

### 8.1 配置加载优先级

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Config Loading Priority                           │
├─────────────────────────────────────────────────────────────────────┤
│  1️⃣ 进程环境变量（最高优先级）                                        │
│     父 shell/daemon 传递给 Gateway 进程                              │
├─────────────────────────────────────────────────────────────────────┤
│  2️⃣ 当前工作目录的 .env                                              │
│     dotenv 默认行为，不覆盖已有值                                     │
├─────────────────────────────────────────────────────────────────────┤
│  3️⃣ 全局 .env ⭐ 推荐                                                │
│     ~/.openclaw/.env                                                │
│     所有实例共享                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  4️⃣ Config env 块                                                   │
│     ~/.openclaw/openclaw.json 中的 env 配置                         │
├─────────────────────────────────────────────────────────────────────┤
│  5️⃣ Login-shell 导入（可选）                                        │
│     env.shellEnv.enabled 或 OPENCLAW_LOAD_SHELL_ENV=1              │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.2 配置文件结构

```json
{
  // Gateway 配置
  "gateway": {
    "port": 18789,
    "bind": "lan",
    "auth": { "mode": "token", "token": "xxx" },
    "controlUi": { "allowedOrigins": [...] }
  },
  
  // Agents 配置
  "agents": {
    "defaults": {
      "model": "bailian/qwen3.5-plus",
      "memorySearch": { "enabled": true },
      "tools": { "allow": ["*"] }
    },
    "list": [
      { "id": "main", "workspace": "..." },
      { "id": "stock", "workspace": "..." }
    ]
  },
  
  // 模型配置
  "models": {
    "providers": {
      "bailian": {
        "baseUrl": "...",
        "apiKey": "${BAILIAN_API_KEY}",
        "api": "openai-completions"
      }
    }
  },
  
  // 频道配置
  "channels": {
    "telegram": { "enabled": true, "accounts": {...} },
    "discord": { "enabled": false }
  },
  
  // 插件配置
  "plugins": {
    "slots": { "memory": "memory-lancedb" },
    "entries": { "memory-lancedb": {...} }
  },
  
  // 绑定配置
  "bindings": [
    { "agentId": "main", "match": { "channel": "telegram", "accountId": "core" } }
  ]
}
```

### 8.3 环境变量规范

**推荐使用全局 .env**:

```bash
# ~/.openclaw/.env

# AI Model Providers
BAILIAN_API_KEY=sk-sp-xxxxxxxxxxxx
OPENAI_API_KEY=sk-xxxxxxxxxxxx
OLLAMA_BASE_URL=http://127.0.0.1:11434/v1

# Telegram Bot Tokens
TELEGRAM_CORE_BOT_TOKEN=xxx
TELEGRAM_STOCK_BOT_TOKEN=xxx

# Network Proxy
HTTP_PROXY=http://127.0.0.1:7897
HTTPS_PROXY=http://127.0.0.1:7897

# Gateway
GATEWAY_AUTH_TOKEN=your-token
```

### 8.4 配置热更新

```python
class ConfigManager:
    def __init__(self, config_file: str):
        self.config_file = config_file
        self.config = {}
        self.watchers = []
    
    async def load(self):
        with open(self.config_file, 'r') as f:
            self.config = json.load(f)
        
        # 解析环境变量引用
        self.resolve_env_vars(self.config)
    
    def resolve_env_vars(self, config: dict):
        """递归解析 ${VAR} 格式的环境变量引用"""
        for key, value in config.items():
            if isinstance(value, str) and value.startswith("${") and value.endswith("}"):
                var_name = value[2:-1]
                config[key] = os.environ.get(var_name, "")
            elif isinstance(value, dict):
                self.resolve_env_vars(value)
    
    async def watch(self):
        """监听配置文件变化"""
        watcher = asyncio.create_task(self._watch_file())
        self.watchers.append(watcher)
    
    async def _watch_file(self):
        last_mtime = os.path.getmtime(self.config_file)
        while True:
            await asyncio.sleep(5)
            current_mtime = os.path.getmtime(self.config_file)
            if current_mtime > last_mtime:
                await self.load()
                await self.notify_change()
                last_mtime = current_mtime
    
    async def notify_change(self):
        # 通知所有配置变更监听者
        ...
```

---

## 九、Memory 系统整合

### 9.1 Memory 架构概览

OpenClaw 提供 **5 种 Memory 插件方案**：

| 插件 | 推荐度 | 检索质量 | 特点 |
|------|--------|---------|------|
| **Memory Core** | ⭐⭐⭐⭐⭐ | 0.72 | 内置、无依赖、Markdown 存储 |
| **QMD** | ⭐⭐⭐⭐ | 0.78 | Reranking、会话索引 |
| **Honcho** | ⭐⭐⭐⭐ | 0.74 | 跨会话、用户建模 |
| **LanceDB Pro** | ⭐⭐⭐⭐ | 0.82 | 自动捕获、智能遗忘 |
| **LanceDB 基础** | ⭐⭐ | 0.70 | 简单向量存储 |

### 9.2 Memory Core 架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Memory Core Architecture                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  数据源 (Source Files)                                               │
│  ~/.openclaw/agents/{agent}/workspace/MEMORY.md                     │
│  ~/.openclaw/agents/{agent}/workspace/memory/*.md                   │
│      ↓ 同步/索引 (自动)                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 向量数据库 (SQLite + sqlite-vec)                              │   │
│  │ ~/.openclaw/memory/{agentId}.sqlite                          │   │
│  │ ├─ chunks (文本分块元数据)                                    │   │
│  │ ├─ chunks_vec (向量索引表)                                    │   │
│  │ └─ chunks_fts* (FTS5 全文搜索索引)                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│      ↓ 查询                                                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ memory_search 工具                                            │   │
│  │ 1. Ollama 生成查询向量                                        │   │
│  │ 2. SQLite 混合搜索 (向量 + FTS)                               │   │
│  │ 3. 加权融合 + MMR 去重                                        │   │
│  │ 4. 返回结果 (带原始文件路径引用)                               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 9.3 混合搜索流程

```python
class MemorySearch:
    async def search(self, query: str, max_results: int = 6) -> List[dict]:
        # 1. 查询向量化
        query_vec = await self.embedder.embed(query)
        
        # 2. 并行搜索
        vector_results = await self.vector_search(query_vec, max_results * 4)
        keyword_results = await self.keyword_search(query, max_results * 4)
        
        # 3. 分数归一化与融合
        results = self.merge_results(
            vector_results, 
            keyword_results,
            vector_weight=0.7,
            text_weight=0.3
        )
        
        # 4. MMR 去重
        results = self.mmr_rerank(results, lambda=0.7)
        
        # 5. 返回结果
        return results[:max_results]
    
    async def vector_search(self, query_vec: List[float], limit: int) -> List[dict]:
        # SQLite sqlite-vec 查询
        sql = """
            SELECT path, start_line, end_line, snippet, 
                   cosine_similarity(vec, ?) as score
            FROM chunks_vec
            ORDER BY score DESC
            LIMIT ?
        """
        return await self.db.execute(sql, [query_vec, limit])
    
    async def keyword_search(self, query: str, limit: int) -> List[dict]:
        # SQLite FTS5 BM25 搜索
        sql = """
            SELECT path, start_line, end_line, snippet, 
                   bm25(chunks_fts) as score
            FROM chunks_fts
            WHERE chunks_fts MATCH ?
            ORDER BY score DESC
            LIMIT ?
        """
        return await self.db.execute(sql, [query, limit])
    
    def merge_results(self, vector_results, keyword_results, 
                      vector_weight: float, text_weight: float) -> List[dict]:
        # 合并去重
        all_results = {}
        
        for r in vector_results:
            r["normalized_score"] = self.normalize(r["score"])
            r["final_score"] = vector_weight * r["normalized_score"]
            all_results[r["path"]] = r
        
        for r in keyword_results:
            r["normalized_score"] = self.normalize(r["score"])
            if r["path"] in all_results:
                all_results[r["path"]]["final_score"] += text_weight * r["normalized_score"]
            else:
                r["final_score"] = text_weight * r["normalized_score"]
                all_results[r["path"]] = r
        
        return sorted(all_results.values(), key=lambda x: x["final_score"], reverse=True)
```

### 9.4 记忆文件组织

```
workspace/
├── MEMORY.md              # 核心长期记忆（COLD）
├── AGENTS.md              # Agent 行为规则
├── SOUL.md                # Agent 人格定义
├── memory/
│   ├── hot/
│   │   └── HOT_MEMORY.md  # 活跃任务（HOT）
│   ├── warm/
│   │   └── WARM_MEMORY.md # 稳定配置（WARM）
│   ├── YYYY-MM-DD.md      # 每日日志
│   ├── projects/          # 项目相关记忆
│   └── archive/           # 归档记忆
└── SESSION-STATE.md       # 会话状态
```

---

## 十、多 Agent 架构

### 10.1 多 Agent 设计原则

| 原则 | 描述 |
|------|------|
| **完全隔离** | 每个代理运行在独立环境中，互不影响 |
| **安全沙箱** | 限制每个代理的权限范围，防止越权操作 |
| **通道独立** | 每个代理可以有独立的 Telegram/Discord Bot |

### 10.2 Agent 角色定义

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "workspace": "~/.openclaw/agents/main/workspace",
        "description": "核心代理 - 系统协调、用户接口",
        "permissions": { "fs": { "allow": ["*"] }, "tools": { "allow": ["*"] } }
      },
      {
        "id": "stock",
        "workspace": "~/.openclaw/agents/stock/workspace",
        "description": "股票代理 - 投资组合监控",
        "permissions": { "fs": { "allow": ["~/.openclaw/agents/stock/**"] } }
      },
      {
        "id": "coding",
        "workspace": "~/.openclaw/agents/coding/workspace",
        "description": "编程代理 - 代码开发、GitHub 管理",
        "permissions": { "tools": { "allow": ["read", "write", "exec", "web_search"] } }
      }
    ]
  }
}
```

### 10.3 Agent 通信协议

**Telegram 多 Bot 方案**:
```
Core Agent: @core_yourname_bot
Stock Agent: @stock_yourname_bot
Coding Agent: @coding_yourname_bot
```

**消息格式**:
```json
{
  "from": "stock",
  "to": "core",
  "action": "portfolio_update",
  "timestamp": "2026-05-11T10:00:00Z",
  "data": { ... }
}
```

### 10.4 Binding 路由规则

```json
{
  "bindings": [
    // Stock Agent - 群组绑定
    {
      "agentId": "stock",
      "match": {
        "channel": "telegram",
        "accountId": "stock",
        "peer": { "kind": "group", "id": "-100xxx" }
      }
    },
    // Stock Agent - 私聊绑定（必须有！）
    {
      "agentId": "stock",
      "match": {
        "channel": "telegram",
        "accountId": "stock"
      }
    }
  ]
}
```

### 10.5 子 Agent 调用

```python
# sessions_spawn 工具实现
async def sessions_spawn(params: dict, client: Client):
    agent_id = params.get("agentId")
    task = params.get("task")
    
    # 1. 创建子 Agent Session
    child_session = await SessionManager.create(agent_id)
    
    # 2. 执行子 Agent
    result = await AgentCore.run(
        agent_id=agent_id,
        messages=[{"role": "user", "content": task}],
        tools=get_tools_for_agent(agent_id)
    )
    
    # 3. 返回结果给父 Agent
    return {
        "agentId": agent_id,
        "result": result,
        "sessionId": child_session.id
    }
```

---

## 十一、部署方案

### 11.1 Docker 沙箱方案（推荐）

```yaml
# docker-compose.yml
version: '3.8'
services:
  main:
    image: openclaw:latest
    container_name: main
    ports:
      - "18789:18789"
    volumes:
      - ~/.openclaw-main:/home/node/.openclaw
    environment:
      - TELEGRAM_BOT_TOKEN=${TELEGRAM_MAIN_BOT_TOKEN}
    restart: unless-stopped

  stock:
    image: openclaw:latest
    container_name: stock
    ports:
      - "9999:18789"
    volumes:
      - ~/.openclaw-stock:/home/node/.openclaw
    environment:
      - TELEGRAM_BOT_TOKEN=${TELEGRAM_STOCK_BOT_TOKEN}
    restart: unless-stopped

  coding:
    image: openclaw:latest
    container_name: coding
    ports:
      - "9998:18789"
    volumes:
      - ~/.openclaw-coding:/home/node/.openclaw
    environment:
      - TELEGRAM_BOT_TOKEN=${TELEGRAM_CODING_BOT_TOKEN}
    restart: unless-stopped
```

### 11.2 原生多进程方案

```bash
#!/bin/bash
# 启动脚本

# Main Agent
OPENCLAW_CONFIG_PATH=~/.openclaw/agents/main/openclaw.json \
openclaw gateway start --workspace ~/.openclaw/agents/main/workspace &

# Stock Agent
OPENCLAW_CONFIG_PATH=~/.openclaw/agents/stock/openclaw.json \
openclaw gateway start --workspace ~/.openclaw/agents/stock/workspace &

# Coding Agent
OPENCLAW_CONFIG_PATH=~/.openclaw/agents/coding/openclaw.json \
openclaw gateway start --workspace ~/.openclaw/agents/coding/workspace &
```

### 11.3 关键配置步骤

```bash
# 1. 正确挂载目录（关键！）
docker run -v ~/.openclaw-stock:/home/node/.openclaw ...  # ✅
docker run -v ~/.openclaw-stock:/root/.openclaw ...        # ❌

# 2. 配置绑定模式
docker exec stock openclaw config set gateway.bind "lan"

# 3. 配置允许的源地址
docker exec stock openclaw config set gateway.controlUi.allowedOrigins '["http://localhost:9999"]'

# 4. 设置认证 Token
docker exec stock openclaw config set gateway.auth.token "your-token"

# 5. 配置模型 Provider
docker exec stock openclaw config set models.providers.bailian '{...}'

# 6. 设置默认模型
docker exec stock openclaw config set agents.defaults.model "bailian/qwen3.5-plus"

# 7. 重启容器
docker restart stock
```

---

## 十二、复刻实施指南

### 12.1 复刻路线图

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Clone Implementation Roadmap                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Phase 1: 核心架构 (Week 1-2)                                        │
│  ├─ L1 Gateway Server                                              │
│  ├─ L2 WebSocket Handler                                           │
│  ├─ L3 Request Router                                              │
│  └─────────────────────────────────────────────────────────────┐   │
│  Phase 2: Agent Core (Week 3-4)                                     │
│  ├─ L5 Agent 执行层                                                │
│  ├─ Tool Call 循环                                                 │
│  ├─ Session 管理                                                   │
│  └─────────────────────────────────────────────────────────────┐   │
│  Phase 3: 工具系统 (Week 5-6)                                        │
│  ├─ 基础工具实现 (read/write/exec)                                  │
│  ├─ 插件系统                                                       │
│  ├─ 工具权限控制                                                   │
│  └─────────────────────────────────────────────────────────────┐   │
│  Phase 4: 通道集成 (Week 7-8)                                        │
│  ├─ Telegram 频道                                                  │
│  ├─ Discord 频道                                                   │
│  ├─ 认证配对                                                       │
│  └─────────────────────────────────────────────────────────────┐   │
│  Phase 5: Memory 系统 (Week 9-10)                                   │
│  ├─ Memory Core                                                    │
│  ├─ 向量索引                                                       │
│  ├─ 混合搜索                                                       │
│  └─────────────────────────────────────────────────────────────┐   │
│  Phase 6: 多 Agent (Week 11-12)                                     │
│  ├─ Agent 隔离                                                     │
│  ├─ Binding 路由                                                   │
│  ├─ 子 Agent 调用                                                  │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 12.2 技术栈选择建议

**Python 版本推荐技术栈**:

| 组件 | 推荐 | 替代方案 |
|------|------|----------|
| **Web Server** | aiohttp | FastAPI, Flask |
| **WebSocket** | websockets | socketio |
| **向量存储** | LanceDB | PostgreSQL + pgvector |
| **全文搜索** | MeiliSearch | Elasticsearch |
| **嵌入模型** | sentence-transformers | OpenAI API |
| **LLM API** | openai SDK | litellm |
| **配置管理** | pydantic | dynaconf |

**核心依赖**:
```python
# requirements.txt
aiohttp>=3.9.0
websockets>=12.0
lancedb>=0.4.0
sentence-transformers>=2.2.0
openai>=1.0.0
pydantic>=2.0.0
python-dotenv>=1.0.0
```

### 12.3 最小化复刻方案

**最小可运行版本**:

```python
# miniclaw.py - 最小化实现
import asyncio
import json
from aiohttp import web
import openai

class MiniClaw:
    def __init__(self, port=18789, model="gpt-4"):
        self.port = port
        self.model = model
        self.clients = {}
        self.tools = [
            {
                "name": "read",
                "description": "Read a file",
                "input_schema": {
                    "type": "object",
                    "properties": {"path": {"type": "string"}},
                    "required": ["path"]
                }
            }
        ]
    
    async def start(self):
        app = web.Application()
        app.router.add_get("/", self.handle_http)
        app.router.add_get("/ws", self.handle_ws)
        runner = web.AppRunner(app)
        await runner.setup()
        site = web.TCPSite(runner, "0.0.0.0", self.port)
        await site.start()
        print(f"MiniClaw running on port {self.port}")
    
    async def handle_ws(self, request):
        ws = web.WebSocketResponse()
        await ws.prepare(request)
        
        async for msg in ws:
            if msg.type == web.WSMsgType.TEXT:
                frame = json.loads(msg.data)
                await self.handle_frame(ws, frame)
        
        return ws
    
    async def handle_frame(self, ws, frame):
        if frame["type"] == "req":
            if frame["method"] == "chat":
                await self.handle_chat(ws, frame)
    
    async def handle_chat(self, ws, frame):
        user_input = frame["params"]["body"]
        
        # Agent 循环
        messages = [{"role": "user", "content": user_input}]
        
        while True:
            response = await openai.ChatCompletion.acreate(
                model=self.model,
                messages=messages,
                tools=self.tools
            )
            
            message = response.choices[0].message
            
            # 发送文本
            if message.content:
                await ws.send_str(json.dumps({
                    "type": "event",
                    "id": frame["id"],
                    "event": {"type": "assistant_text", "text": message.content}
                }))
            
            # 处理工具调用
            if message.tool_calls:
                for tool_call in message.tool_calls:
                    result = await self.execute_tool(
                        tool_call.function.name,
                        json.loads(tool_call.function.arguments)
                    )
                    messages.append({"role": "assistant", "content": None, "tool_calls": [tool_call]})
                    messages.append({"role": "tool", "tool_call_id": tool_call.id, "content": result})
                continue
            
            # 结束
            await ws.send_str(json.dumps({
                "type": "res",
                "id": frame["id"],
                "ok": True,
                "result": {"content": message.content}
            }))
            break
    
    async def execute_tool(self, name: str, params: dict) -> str:
        if name == "read":
            try:
                with open(params["path"], "r") as f:
                    return f.read()
            except Exception as e:
                return f"Error: {str(e)}"
        return "Unknown tool"

if __name__ == "__main__":
    claw = MiniClaw()
    asyncio.run(claw.start())
```

### 12.4 关键实现要点

| 要点 | OpenClaw 实现 | 复刻建议 |
|------|---------------|----------|
| **WebSocket 消息** | JSON frame | 使用统一的消息格式 |
| **Tool Call 循环** | while loop + continue | 必须正确处理 tool_result |
| **插件加载** | slots + entries | 使用依赖注入模式 |
| **Session 管理** | 文件持久化 | SQLite 或文件系统 |
| **Compaction** | 摘要压缩 | 定期检查 token 数 |
| **认证配对** | device + token | 最简版本可跳过 |
| **权限控制** | allow/deny 列表 | 最简版本可跳过 |

### 12.5 测试验证清单

```
□ Phase 1 验证
  □ WebSocket 连接成功
  □ JSON frame 解析正确
  □ 路由分发正确

□ Phase 2 验证
  □ Agent 能回复文本
  □ Tool Call 循环正常
  □ Session 持久化正确

□ Phase 3 验证
  □ read/write/exec 工具可用
  □ 插件加载正确
  □ 权限控制生效

□ Phase 4 验证
  □ Telegram Bot 连接成功
  □ 消息收发正常
  □ 配对流程正确

□ Phase 5 验证
  □ memory_search 返回结果
  □ 向量索引正常
  □ 文件同步正确

□ Phase 6 验证
  □ 多 Agent 隔离正确
  □ Binding 路由正常
  □ 子 Agent 调用成功
```

---

## 附录

### A. 完整配置示例

```json
{
  "gateway": {
    "port": 18789,
    "bind": "lan",
    "auth": {
      "mode": "token",
      "token": "your-secret-token"
    },
    "controlUi": {
      "allowedOrigins": ["http://localhost:18789"]
    }
  },
  
  "agents": {
    "defaults": {
      "model": "bailian/qwen3.5-plus",
      "memorySearch": {
        "enabled": true,
        "provider": "ollama",
        "sync": { "watch": true, "watchDebounceMs": 1500 },
        "query": {
          "maxResults": 8,
          "hybrid": {
            "enabled": true,
            "vectorWeight": 0.7,
            "textWeight": 0.3,
            "candidateMultiplier": 4,
            "mmr": { "enabled": true, "lambda": 0.7 }
          }
        }
      },
      "tools": {
        "allow": ["*"],
        "deny": ["exec:rm -rf", "exec:sudo"]
      },
      "maxConcurrent": 4,
      "timeout": 120
    },
    "list": [
      {
        "id": "main",
        "workspace": "~/.openclaw/agents/main/workspace",
        "heartbeat": {
          "every": "4h",
          "activeHours": { "start": "08:00", "end": "23:00", "timezone": "Asia/Shanghai" }
        }
      },
      {
        "id": "stock",
        "workspace": "~/.openclaw/agents/stock/workspace",
        "tools": {
          "allow": ["read", "web_search", "memory_search"],
          "deny": ["exec", "write"]
        }
      }
    ]
  },
  
  "models": {
    "providers": {
      "bailian": {
        "baseUrl": "https://coding.dashscope.aliyuncs.com/v1",
        "apiKey": "${BAILIAN_API_KEY}",
        "api": "openai-completions",
        "models": [
          {
            "id": "qwen3.5-plus",
            "name": "qwen3.5-plus",
            "api": "openai-completions",
            "contextWindow": 1000000,
            "maxTokens": 65536
          }
        ]
      }
    }
  },
  
  "channels": {
    "telegram": {
      "enabled": true,
      "proxy": "http://127.0.0.1:7897",
      "accounts": {
        "core": {
          "botToken": "${TELEGRAM_CORE_BOT_TOKEN}",
          "dmPolicy": "pairing",
          "groupPolicy": "allowlist",
          "allowFrom": [123456789]
        },
        "stock": {
          "botToken": "${TELEGRAM_STOCK_BOT_TOKEN}",
          "dmPolicy": "pairing"
        }
      }
    }
  },
  
  "plugins": {
    "slots": {
      "memory": "memory-core",
      "search": "brave"
    },
    "entries": {
      "brave": {
        "enabled": true,
        "config": {
          "apiKey": "${BRAVE_API_KEY}"
        }
      }
    }
  },
  
  "bindings": [
    {
      "agentId": "main",
      "match": { "channel": "telegram", "accountId": "core" }
    },
    {
      "agentId": "stock",
      "match": { "channel": "telegram", "accountId": "stock" }
    }
  ]
}
```

### B. 环境变量模板

```bash
# ~/.openclaw/.env

# AI Model Providers
BAILIAN_API_KEY=sk-sp-xxxxxxxxxxxx
OPENAI_API_KEY=sk-xxxxxxxxxxxx
OLLAMA_BASE_URL=http://127.0.0.1:11434/v1

# Telegram Bot Tokens
TELEGRAM_CORE_BOT_TOKEN=xxx
TELEGRAM_STOCK_BOT_TOKEN=xxx
TELEGRAM_CODING_BOT_TOKEN=xxx

# Search API
BRAVE_API_KEY=xxx

# GitHub
GH_TOKEN=gho_xxxxxxxxxx

# Network Proxy
HTTP_PROXY=http://127.0.0.1:7897
HTTPS_PROXY=http://127.0.0.1:7897
NO_PROXY=localhost,127.0.0.1

# Gateway
GATEWAY_AUTH_TOKEN=your-secret-token

# Logging
OPENCLAW_LOG_LEVEL=info
```

### C. 参考资源

- **OpenClaw 官方文档**: https://docs.openclaw.ai
- **OpenClaw GitHub**: https://github.com/openclaw/openclaw
- **sqlite-vec**: https://github.com/asg017/sqlite-vec
- **LanceDB**: https://lancedb.com
- **LangGraph 对比**: /docs/langgraph-technical-report/

---

## 版本历史

| 版本 | 日期 | 更新内容 |
|------|------|---------|
| 1.0 | 2026-05-11 | 初始完整版本 |

---

*本文档基于 OpenClaw 官方架构和 ai-note 项目实践经验编写，为复刻类似系统提供完整指导。*