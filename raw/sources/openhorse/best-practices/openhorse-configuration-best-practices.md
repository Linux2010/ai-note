# OpenHorse 配置与最佳实践

> **OpenHorse — 通用 Agent 驾驭框架**
> 版本：v0.1.21 | 作者：Linux2010 | 许可：MIT

---

## 目录

1. [项目概览](#项目概览)
2. [安装与初始化](#安装与初始化)
3. [核心配置详解](#核心配置详解)
4. [模型选择与切换](#模型选择与切换)
5. [MCP 协议配置](#mcp-协议配置)
6. [记忆系统配置](#记忆系统配置)
7. [上下文管理最佳实践](#上下文管理最佳实践)
8. [安全与权限配置](#安全与权限配置)
9. [Web 搜索配置](#web-搜索配置)
10. [会话管理](#会话管理)
11. [常见问题与排错](#常见问题与排错)
12. [生产环境推荐配置](#生产环境推荐配置)

---

## 项目概览

OpenHorse 是一个 CLI 驱动的编码 Agent 框架，核心理念是 **"AI 如马，OpenHorse 如缰"**——强大的模型需要引导和约束。

### 核心能力

| 能力 | 说明 |
|------|------|
| **工具编排** | 20+ 内置工具：文件读写、搜索、Shell、网页、记忆、任务、计划 |
| **多模型支持** | OpenAI、Claude、DashScope（GLM/Qwen/Kimi）、自定义端点 |
| **上下文感知** | 每个模型独立的上下文窗口，95% 自动压缩 |
| **MCP 协议** | 完整 MCP 支持，心跳检测 + 指数退避重连 |
| **记忆系统** | 用户 / 项目 / 会话记忆，支持语义搜索 |
| **安全边界** | Bash 安全检查、审计日志、权限模式 |

### 技术栈

- **语言**：TypeScript 5.x
- **运行时**：Node.js >= 18.0
- **LLM SDK**：OpenAI SDK（兼容多服务商）
- **向量存储**：SQLite vec0（语义搜索）
- **UI**：React Reconciler + Ink（终端渲染）

---

## 安装与初始化

### 全局安装（推荐）

```bash
npm install -g openhorse

# 验证安装
openhorse --version
# 输出: v0.1.21
```

### 从源码安装

```bash
git clone https://github.com/Linux2010/openhorse.git
cd openhorse
npm install
npm run build
npm link

# 任意目录运行
openhorse
```

### 首次启动

首次运行 `openhorse` 时，会自动创建 `~/.openhorse/` 目录结构：

```
~/.openhorse/
├── openhorse.json       # 用户配置（核心）
├── mcp.json             # MCP Server 配置
├── sessions/            # 会话持久化
├── projects/            # 项目级记忆
├── cache/               # 缓存
├── cost/                # 成本记录
└── input-history.json   # 输入历史
```

---

## 核心配置详解

### 配置文件位置

`~/.openhorse/openhorse.json`

### 用户可配置字段（4 个核心字段）

```json
{
  "apiKey": "sk-xxx",
  "apiBaseUrl": "https://api.openai.com/v1",
  "defaultModel": "gpt-4o",
  "fallbackModel": "gpt-4o-mini"
}
```

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `apiKey` | ✅ | - | LLM API 密钥 |
| `apiBaseUrl` | ❌ | OpenAI 官方 | API 端点 URL |
| `defaultModel` | ❌ | `glm-5` | 默认使用的模型 |
| `fallbackModel` | ❌ | - | 主模型失败时的降级模型 |

### Agent 内部参数（自动管理）

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `maxTokens` | 8192 | 最大输出 Token 数 |
| `temperature` | 0.1 | 采样温度（编码任务建议低值） |
| `maxRetries` | 3 | 失败重试次数 |
| `retryBaseDelay` | 1000ms | 重试基础延迟（指数退避） |

> ⚠️ **注意**：`maxTokens`、`temperature`、`maxRetries` 等参数由 Agent 内部管理，用户无需手动配置。Agent 会根据任务类型自动调整。

### 配置优先级

```
CLI 参数 > ~/.openhorse/openhorse.json > 环境变量 > 内部默认值
```

### 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `OPENHORSE_API_KEY` | - | LLM API 密钥 |
| `OPENHORSE_API_BASE_URL` | - | API 基础 URL |
| `OPENHORSE_MODEL` | `glm-5` | 默认模型 |
| `OPENHORSE_MODE` | `development` | 运行模式 |
| `OPENHORSE_LOG_LEVEL` | `info` | 日志级别 |
| `OPENHORSE_EMBEDDING_PROVIDER` | - | Embedding 服务（ollama/openai） |
| `OPENHORSE_WEBSEARCH_API_KEY` | - | Web 搜索 API Key |
| `OPENHORSE_WEBSEARCH_PROVIDER` | - | Web 搜索提供商（ddg/tavily/brave/custom/zhipu） |

---

## 模型选择与切换

### 支持的模型家族

| 服务商 | 推荐模型 | 端点 | 上下文窗口 | 最大输出 |
|--------|----------|------|-----------|----------|
| **GLM（智谱）** | `glm-5` | DashScope coding | 202,752 | 8,192 |
| **Qwen（通义）** | `qwen-plus` | DashScope coding | 131,072 | 8,192 |
| **Qwen Long** | `qwen-long` | DashScope coding | 1,000,000 | 8,192 |
| **OpenAI** | `gpt-4o` | OpenAI API | 128,000 | 16,384 |
| **OpenAI Mini** | `gpt-4o-mini` | OpenAI API | 128,000 | 16,384 |
| **Claude Sonnet** | `claude-sonnet-4-6` | Anthropic API | 200,000 | 16,000 |
| **Claude Opus** | `claude-opus-4-8` | Anthropic API | 200,000 | 32,000 |
| **DeepSeek** | `deepseek-chat` | DeepSeek API | 128,000 | 8,192 |

### 模型选择建议

| 场景 | 推荐模型 | 原因 |
|------|----------|------|
| **日常编码** | `glm-5` / `gpt-4o` | 平衡性能与成本 |
| **长上下文任务** | `qwen-long` / `claude-sonnet-4-6` | 100 万 / 20 万上下文 |
| **复杂推理** | `claude-opus-4-8` / `deepseek-reasoner` | 最强推理能力 |
| **低成本批量** | `gpt-4o-mini` / `qwen-turbo` | 最低成本 |
| **中文场景** | `glm-5` / `qwen-plus` | 中文优化 |

### 运行时切换模型

```bash
# 查看当前模型
/model

# 列出所有可用模型
/model list

# 切换模型
/model glm-5
/model claude-sonnet-4-6
```

### 配置 Fallback 模型

```json
{
  "defaultModel": "glm-5",
  "fallbackModel": "qwen-plus"
}
```

当主模型请求失败时，自动降级到 `fallbackModel`，确保任务不中断。

### 自定义 API 端点

OpenHorse 兼容任何 OpenAI API 格式的端点：

```json
{
  "apiKey": "your-key",
  "apiBaseUrl": "https://your-proxy.com/v1",
  "defaultModel": "your-model-name"
}
```

**常见端点配置**：

| 服务商 | apiBaseUrl |
|--------|------------|
| OpenAI 官方 | `https://api.openai.com/v1` |
| DashScope（阿里） | `https://dashscope.aliyuncs.com/compatible-mode/v1` |
| DashScope Coding | `https://coding.dashscope.aliyuncs.com/v1` |
| DeepSeek | `https://api.deepseek.com/v1` |
| 智谱 GLM | `https://open.bigmodel.cn/api/paas/v4` |
| 自建代理 | `https://your-proxy.com/v1` |

---

## MCP 协议配置

### 什么是 MCP

MCP (Model Context Protocol) 允许 OpenHorse 连接外部工具服务器，扩展 Agent 能力。

### 配置文件

`~/.openhorse/mcp.json`

### 基础配置模板

```json
{
  "servers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-filesystem", "/allowed/dir"],
      "env": {}
    }
  }
}
```

### 常用 MCP Server 配置

#### 文件系统访问

```json
{
  "filesystem": {
    "command": "npx",
    "args": ["-y", "@anthropic/mcp-server-filesystem", "/home/user/projects"],
    "env": {}
  }
}
```

#### Telegram Bot

```json
{
  "telegram": {
    "command": "node",
    "args": ["path/to/plugin-telegram/dist/index.js"],
    "env": {
      "TELEGRAM_BOT_TOKEN": "your-token"
    }
  }
}
```

#### GitHub

```json
{
  "github": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-github"],
    "env": {
      "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxx"
    }
  }
}
```

#### PostgreSQL

```json
{
  "postgres": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://user:pass@localhost/db"],
    "env": {}
  }
}
```

### MCP 最佳实践

1. **最小权限原则**：只允许访问必要的目录和资源
2. **环境变量隔离**：敏感信息（Token、密码）放在 `env` 字段，不要硬编码
3. **心跳检测**：OpenHorse 自动进行心跳检测，无需手动配置
4. **重连策略**：内置指数退避重连，断线后自动恢复
5. **查看状态**：使用 `/mcp` 命令查看所有 MCP Server 连接状态

---

## 记忆系统配置

### 记忆层级

OpenHorse 的记忆系统分为 4 层：

| 层级 | 存储位置 | 生命周期 | 用途 |
|------|----------|----------|------|
| **工作记忆** | 对话上下文 | 当前会话 | 当前任务信息 |
| **短期记忆** | 会话文件 | 会话期间 | 会话级上下文 |
| **长期记忆** | `~/.openhorse/` | 永久 | 用户偏好、项目知识 |
| **语义搜索** | SQLite vec0 | 永久 | 向量检索历史记忆 |

### 记忆类型

| 类型 | 说明 | 示例 |
|------|------|------|
| `user` | 用户偏好 | "我喜欢用 TypeScript" |
| `feedback` | 反馈规则 | "代码风格：使用函数式" |
| `project` | 项目知识 | "本项目使用 pnpm" |
| `reference` | 参考资料 | "API 文档链接" |

### 记忆命令

```bash
# 查看记忆状态
/memory

# 重建语义搜索索引
/memory reindex
```

### 记忆最佳实践

1. **主动保存关键决策**：让 Agent 记住项目约定和技术选型
2. **使用语义搜索**：`memory_recall` 支持模糊匹配，无需精确关键词
3. **定期 reindex**：大量记忆后执行 `/memory reindex` 优化搜索
4. **分类存储**：使用 `user`/`project`/`feedback`/`reference` 四种类型分类
5. **及时清理**：过时记忆用 `memory_forget` 删除

### Embedding 配置

语义搜索需要 Embedding 服务，支持两种方式：

```bash
# 方式 1：Ollama（本地，免费）
export OPENHORSE_EMBEDDING_PROVIDER=ollama

# 方式 2：OpenAI（云端，付费）
export OPENHORSE_EMBEDDING_PROVIDER=openai
export OPENAI_API_KEY=sk-xxx
```

---

## 上下文管理最佳实践

### 自动压缩机制

OpenHorse 在上下文使用量达到 **95%** 时自动压缩：

1. LLM 对早期消息生成摘要
2. 用 `[Context Summary]` 块替代旧消息
3. 保留系统消息和最近消息
4. 状态栏显示压缩信息

```
Compact: 30 → 8 messages | Context: 45% → 12%
```

### 手动压缩

```bash
/compact    # 手动触发上下文压缩（不受 30 秒间隔限制）
```

### 上下文管理建议

| 建议 | 说明 |
|------|------|
| **关注状态栏** | 实时显示 Token 用量和上下文百分比 |
| **及时压缩** | 上下文 > 80% 时主动 `/compact` |
| **选择合适模型** | 长任务用 `qwen-long`（100 万上下文） |
| **拆分任务** | 复杂任务拆分为多个会话，避免单会话过长 |
| **利用记忆** | 重要信息存入记忆系统，压缩后仍可检索 |

---

## 安全与权限配置

### 工具确认模式

```json
{
  "toolConfirmation": "allow"
}
```

| 模式 | 说明 |
|------|------|
| `allow` | 自动批准所有工具调用（开发环境推荐） |
| `confirm` | 每次工具调用需确认（生产环境推荐） |
| `config` | 根据工具类型决定（默认） |

### Bash 安全检查

OpenHorse 内置 Bash 命令安全检查，会拦截危险命令：

- `rm -rf /` — 拒绝
- `sudo` 操作 — 需确认
- 管道注入 — 检测并警告

### 安全最佳实践

1. **生产环境用 `confirm` 模式**：避免误操作
2. **限制文件系统访问**：MCP Server 只授权必要目录
3. **API Key 安全**：使用环境变量而非硬编码
4. **审计日志**：OpenHorse 自动记录所有工具调用

---

## Web 搜索配置

### 配置搜索提供商

OpenHorse 支持多种 Web 搜索后端：

```bash
# 方式 1：智谱搜索（默认）
export OPENHORSE_WEBSEARCH_PROVIDER=zhipu
export ZHIPU_API_KEY=your-key

# 方式 2：DuckDuckGo（免费，无需 Key）
export OPENHORSE_WEBSEARCH_PROVIDER=ddg

# 方式 3：Tavily
export OPENHORSE_WEBSEARCH_PROVIDER=tavily
export TAVILY_API_KEY=tvly-xxx

# 方式 4：Brave
export OPENHORSE_WEBSEARCH_PROVIDER=brave
export BRAVE_API_KEY=BSA-xxx

# 方式 5：自定义
export OPENHORSE_WEBSEARCH_PROVIDER=custom
export OPENHORSE_WEBSEARCH_API_KEY=your-key
export OPENHORSE_WEBSEARCH_ENDPOINT=https://your-search-api.com/search
```

### 搜索排错

如果搜索失败，检查：

1. API Key 是否正确
2. 网络是否可达搜索端点
3. 尝试切换提供商（如从 `zhipu` 切换到 `ddg`）
4. 查看 Agent 输出的错误信息中的 suggestion

---

## 会话管理

### 会话命令

```bash
/sessions          # 列出最近会话
/resume            # 恢复上次会话
/clear-history     # 清除当前对话历史
/reset             # 同 clear-history
```

### 会话持久化

会话自动保存到 `~/.openhorse/sessions/`，每个会话一个 JSON 文件。

### 项目级记忆

OpenHorse 为每个工作目录创建独立的项目记忆空间：

```
~/.openhorse/projects/
├── Users-yike-ai-note/     # /Users/yike/ai-note 项目的记忆
├── Users-yike/             # /Users/yike 目录的记忆
└── 19bf81b95ff35060/       # 其他项目（路径哈希）
```

---

## 常见问题与排错

### Q: Web 搜索不可用

**症状**：`WEBSEARCH_UNAVAILABLE` 错误

**解决方案**：

```bash
# 1. 检查当前提供商
echo $OPENHORSE_WEBSEARCH_PROVIDER

# 2. 切换到 DuckDuckGo（免费，无需 Key）
export OPENHORSE_WEBSEARCH_PROVIDER=ddg

# 3. 或配置智谱搜索
export OPENHORSE_WEBSEARCH_PROVIDER=zhipu
export ZHIPU_API_KEY=your-valid-key
```

### Q: 上下文压缩过于频繁

**解决方案**：

1. 切换到更大上下文的模型（如 `qwen-long`）
2. 手动 `/compact` 在合适时机压缩
3. 拆分任务到多个会话

### Q: MCP Server 连接失败

**解决方案**：

1. 使用 `/mcp` 查看连接状态
2. 检查 `mcp.json` 中的路径和命令
3. 确认 MCP Server 依赖已安装
4. OpenHorse 会自动重连（指数退避）

### Q: 模型切换后上下文丢失

**说明**：不同模型有不同的上下文窗口大小，切换模型后 OpenHorse 会自动适配。如果新模型上下文更小，可能触发自动压缩。

### Q: API Key 安全

**最佳实践**：

```bash
# ✅ 推荐：环境变量
export OPENHORSE_API_KEY=sk-xxx

# ✅ 推荐：配置文件（权限 600）
chmod 600 ~/.openhorse/openhorse.json

# ❌ 避免：在代码中硬编码
```

---

## 生产环境推荐配置

### 配置文件 `~/.openhorse/openhorse.json`

```json
{
  "apiKey": "sk-xxx",
  "apiBaseUrl": "https://api.openai.com/v1",
  "defaultModel": "gpt-4o",
  "fallbackModel": "gpt-4o-mini",
  "toolConfirmation": "confirm",
  "ui": {
    "renderer": "legacy",
    "confirmations": "config"
  },
  "maxTokens": 16384,
  "temperature": 0.1
}
```

### MCP 配置 `~/.openhorse/mcp.json`

```json
{
  "servers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-filesystem", "/workspace"],
      "env": {}
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxx"
      }
    }
  }
}
```

### 环境变量 `.env` 或 shell profile

```bash
# API 配置
export OPENHORSE_API_KEY=sk-xxx
export OPENHORSE_API_BASE_URL=https://api.openai.com/v1

# Web 搜索
export OPENHORSE_WEBSEARCH_PROVIDER=ddg

# Embedding（语义搜索）
export OPENHORSE_EMBEDDING_PROVIDER=ollama

# 日志
export OPENHORSE_LOG_LEVEL=warn
export OPENHORSE_MODE=production
```

### 不同场景推荐

| 场景 | defaultModel | fallbackModel | toolConfirmation | temperature |
|------|-------------|---------------|------------------|-------------|
| **个人开发** | `glm-5` | `qwen-plus` | `allow` | `0.1` |
| **团队协作** | `gpt-4o` | `gpt-4o-mini` | `confirm` | `0.1` |
| **代码审查** | `claude-sonnet-4-6` | `gpt-4o` | `confirm` | `0.0` |
| **文档生成** | `qwen-long` | `glm-5` | `allow` | `0.3` |
| **生产部署** | `gpt-4o` | `qwen-plus` | `confirm` | `0.0` |

---

## 交互命令速查

| 命令 | 别名 | 说明 |
|------|------|------|
| `/help` | `/h` | 显示帮助 |
| `/status` | `/s` | 系统状态 |
| `/model` | - | 查看/切换模型 |
| `/config` | - | 显示配置 |
| `/cost` | - | Token 用量和成本 |
| `/compact` | - | 手动压缩上下文 |
| `/sessions` | - | 列出会话 |
| `/resume` | - | 恢复上次会话 |
| `/memory` | - | 记忆状态 |
| `/memory reindex` | - | 重建搜索索引 |
| `/skills` | - | 已加载技能 |
| `/mcp` | - | MCP 状态 |
| `/agents` | - | Agent 列表 |
| `/safety` | - | 安全配置 |
| `/task` | - | 任务管理 |
| `/run` | - | Agent 执行任务 |
| `/clear` | - | 清屏 |
| `/clear-history` | `/reset` | 清除对话 |
| `/exit` | `/q` | 退出 |

---

## 版本更新

```bash
# 更新到最新版
npm update -g openhorse

# 查看当前版本
openhorse --version

# 启用 UI v2 预览
openhorse --ui v2
```

### 版本历史摘要

| 版本 | 关键特性 |
|------|----------|
| v0.1.21 | 当前最新，稳定性提升 |
| v0.1.20 | 测试改进 |
| v0.1.16 | Token 自动压缩、模型上下文感知、动态发现 |
| v0.1.15 | Markdown 流式渲染、CJK 修复 |
| v0.1.14 | LSP 修复、紧凑 UI |
| v0.1.10-13 | MCP 客户端、语义搜索、技能系统 |
| v0.1.1-9 | CLI 框架、Harness、记忆、会话、工具 |

---

## 参考链接

- **GitHub**：https://github.com/Linux2010/openhorse
- **npm**：https://www.npmjs.com/package/openhorse
- **Issues**：https://github.com/Linux2010/openhorse/issues
- **许可证**：MIT

---

*最后更新：2026-06-17 | 基于 OpenHorse v0.1.21*