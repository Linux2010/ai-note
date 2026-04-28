# OpenTask 容器配置指南

> 基于实际经验总结，适用于 Docker 容器环境下的 OpenTask 任务调度系统配置。

---

## 一、环境变量配置

### 1. 必需的环境变量

```bash
# 在 .env 文件或容器启动时配置
OPENTASK_HOST=http://host.docker.internal:8090    # ⚠️ 必须用 host.docker.internal
OPENTASK_API_KEY=your-api-key
OPENTASK_BOT_NAME=your-bot-name
```

### 2. 关键注意事项

| 配置项 | ❌ 错误值 | ✅ 正确值 | 说明 |
|--------|----------|----------|------|
| **OPENTASK_HOST** | `http://127.0.0.1:8090` | `http://host.docker.internal:8090` | 容器内访问宿主机服务必须用 `host.docker.internal` |

**原因**: Docker 容器内的 `127.0.0.1` 或 `localhost` 指向容器本身，而非宿主机。要访问宿主机上运行的服务（如 OpenTask API），必须使用 `host.docker.internal`。

---

## 二、API 接口规范

### 1. 查询待执行任务

```bash
curl -s --noproxy '*' \
  -H 'X-Bot-Key: $OPENTASK_API_KEY' \
  '$OPENTASK_HOST/api/tasks/pending?assigned_to=$OPENTASK_BOT_NAME'
```

### 2. 开始执行任务

```bash
curl -s --noproxy '*' -X PUT \
  -H 'X-Bot-Key: $OPENTASK_API_KEY' \
  '$OPENTASK_HOST/api/tasks/{task_id}/start'
```

### 3. 完成任务

```bash
curl -s --noproxy '*' -X PUT \
  -H 'X-Bot-Key: $OPENTASK_API_KEY' \
  '$OPENTASK_HOST/api/tasks/{task_id}/complete'
```

### 4. 任务失败

```bash
curl -s --noproxy '*' -X PUT \
  -H 'X-Bot-Key: $OPENTASK_API_KEY' \
  '$OPENTASK_HOST/api/tasks/{task_id}/fail'
```

---

## 三、Cron 定时任务配置

### 1. 正确配置模板

```json
{
  "name": "OpenTask 任务检查",
  "schedule": {
    "kind": "every",
    "everyMs": 14400000
  },
  "sessionTarget": "isolated",
  "wakeMode": "now",
  "payload": {
    "kind": "agentTurn",
    "message": "执行 OpenTask 任务检查。\n\n注意：不要输出 HEARTBEAT_OK，否则系统会认为这是心跳确认而不送达报告。",
    "timeoutSeconds": 60
  },
  "delivery": {
    "mode": "announce",
    "channel": "telegram",
    "to": "用户ID"
  }
}
```

### 2. ⚠️ 最关键规则：不要输出 HEARTBEAT_OK

**这是导致报告不送达的根本原因！**

| 输出内容 | 结果 |
|----------|------|
| 包含 `HEARTBEAT_OK` | ❌ 系统视为心跳确认 → **不送达报告** |
| 不包含 `HEARTBEAT_OK` | ✅ 正常输出 → **送达成功** |

**示例对比**:

❌ **错误输出** (不送达):
```
🇺🇸 Heartbeat 检查
- 时间: 2026-04-26 11:07
- API: OK
HEARTBEAT_OK
```

✅ **正确输出** (送达):
```
🇺🇸 OpenTask 检查
时间: 2026-04-26 11:07 (上海)
API: OK
待执行: 0 | Running: 0
```

### 3. 配置参数详解

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| **sessionTarget** | `isolated` | 在隔离会话运行，不依赖当前会话状态 |
| **payload.kind** | `agentTurn` | 运行 agent 执行任务 |
| **payload.timeoutSeconds** | `60` | 合理超时，避免过长导致卡死 |
| **delivery.mode** | `announce` | 送达执行结果给用户 |
| **delivery.channel** | `telegram` | 固定频道（不要用 `"last"`） |
| **delivery.to** | 用户ID | 明确指定接收者 |

### 4. ❌ 常见错误配置

| 错误配置 | 问题 |
|----------|------|
| `channel: "last"` | 最后活跃会话关闭后无法送达 |
| `sessionTarget: "current"` | 当前会话不活跃时无法工作 |
| `timeoutSeconds: 120+` | 可能超时失败 |
| 输出包含 `HEARTBEAT_OK` | 报告不送达 |

---

## 四、任务执行流程

### 1. 标准流程

```
┌─────────────────────────────────────────────────────┐
│  1. 查询待执行任务 (/api/tasks/pending)             │
├─────────────────────────────────────────────────────┤
│  2. 如果有任务:                                     │
│     - 选择优先级最高的 (P0 > P1 > P2)               │
│     - 开始执行: /api/tasks/{id}/start               │
│     - 执行任务内容                                  │
│     - 完成后: /api/tasks/{id}/complete              │
│     - 或失败: /api/tasks/{id}/fail                  │
├─────────────────────────────────────────────────────┤
│  3. 如果没有任务:                                   │
│     - 输出简洁报告（不要 HEARTBEAT_OK）             │
│     - delivery 会自动送达给用户                     │
└─────────────────────────────────────────────────────┘
```

### 2. 任务优先级

- **P0**: 最高优先级，立即执行
- **P1**: 中等优先级
- **P2**: 低优先级

---

## 五、opentask-client 技能配置（建议）

如果需要创建一个技能来管理 OpenTask，建议结构如下：

```
skills/opentask-client/
├── SKILL.md          # 技能说明和触发条件
├── scripts/
│   ├── check-tasks.sh    # 查询待执行任务
│   ├── start-task.sh     # 开始任务
│   └── complete-task.sh  # 完成任务
└── references/
    └── api-spec.md       # API 规范文档
```

### SKILL.md 示例

```markdown
# opentask-client

OpenTask 任务管理系统客户端技能。

## 触发条件

- 用户提到 "OpenTask"、"任务检查"、"待执行任务"
- 定时任务需要检查 OpenTask

## 环境要求

- OPENTASK_HOST=http://host.docker.internal:8090
- OPENTASK_API_KEY 已配置
- OPENTASK_BOT_NAME 已配置

## 执行流程

1. 调用 scripts/check-tasks.sh 查询待执行任务
2. 按优先级执行任务
3. 报告结果（不要输出 HEARTBEAT_OK）
```

---

## 六、完整配置清单

### ✅ 检查项

| 检查项 | 状态 |
|--------|------|
| OPENTASK_HOST 使用 `host.docker.internal` | ✅ |
| OPENTASK_API_KEY 已配置 | ✅ |
| OPENTASK_BOT_NAME 已配置 | ✅ |
| Cron 任务 sessionTarget = `isolated` | ✅ |
| delivery.channel 明确指定 | ✅ |
| delivery.to 明确指定 | ✅ |
| payload 不要求输出 HEARTBEAT_OK | ✅ |
| timeoutSeconds ≤ 60 | ✅ |

---

## 七、故障排查

### 1. 报告不送达

**检查步骤**:
```bash
# 查看最近运行记录
cron action=runs jobId=<job_id>

# 检查 deliveryStatus
# 如果显示 "not-delivered"，检查输出是否包含 HEARTBEAT_OK
```

### 2. API 连接失败

**检查步骤**:
```bash
# 测试连接
curl -s --noproxy '*' \
  -H 'X-Bot-Key: $OPENTASK_API_KEY' \
  'http://host.docker.internal:8090/api/tasks/pending?assigned_to=trump'

# 如果失败，检查：
# 1. OPENTASK_HOST 是否正确
# 2. 宿主机服务是否运行
# 3. 网络是否连通
```

### 3. 任务超时

**解决方案**:
- 减少 timeoutSeconds
- 简化 payload.message
- 检查模型响应速度

---

## 八、最佳实践总结

1. **容器访问宿主机**: 必须用 `host.docker.internal`
2. **Cron 报告送达**: 绝对不要输出 `HEARTBEAT_OK`
3. **delivery 配置**: 明确指定 channel 和 to
4. **sessionTarget**: 使用 `isolated`
5. **超时设置**: 合理值 60 秒
6. **任务流程**: start → 执行 → complete/fail

---

_文档版本: 2026-04-28_
_适用环境: Docker 容器 + OpenClaw + OpenTask_