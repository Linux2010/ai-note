# 群组对话配置最佳实践

> **文档类型**: 最佳实践  
> **适用场景**: OpenClaw 在 Telegram/Discord 等群组中作为 AI 代理参与对话  
> **最后更新**: 2026-05-17  
> **版本**: 1.0.0

---

## 📋 目录

1. [概述](#概述)
2. [核心原则](#核心原则)
3. [群组类型与策略](#群组类型与策略)
4. [配置步骤](#配置步骤)
5. [行为准则](#行为准则)
6. [消息处理最佳实践](#消息处理最佳实践)
7. [故障排查](#故障排查)
8. [高级配置](#高级配置)

---

## 概述

群组对话配置决定了 AI 代理如何在多人聊天环境中表现。与私聊不同，群组需要特殊的策略来确保：
- 不干扰正常的人类交流
- 在需要时恰当地参与
- 遵守社交礼仪和边界

### 群组 vs 私聊的区别

| 维度 | 私聊 | 群组 |
|------|------|------|
| **响应触发** | 所有消息 | 仅 @提及 或明确请求 |
| **响应频率** | 每条消息都回复 | 按需回复，保持克制 |
| **内容范围** | 个人助手功能 | 群组有益的内容 |
| **隐私边界** | 用户个人数据 | 不暴露个人隐私 |
| **角色定位** | 个人助理 | 群组成员/观察者 |

---

## 核心原则

### 1. 参与而非主导 (Participate, Don't Dominate)

**不要响应每条消息**。人类在群聊中也不会这样做。

```
✅ 正确：质量优先，只在能增值时发言
❌ 错误：每条消息都回复（刷屏）
```

### 2. 知道何时说话 (Know When to Speak)

**应该响应的场景**：
- 被 @提及 或直接称呼
- 能提供有价值的信息或见解
- 有幽默/风趣的自然插入机会
- 需要纠正重要的错误信息
- 被要求总结或解释时

**应该保持沉默的场景**：
- 人类的闲聊打趣
- 问题已被其他人回答
- 只能回复 "是啊" 或 "不错"
- 对话本身流畅，不需要介入
- 回复会打断当前的氛围

### 3. 反应像人类 (React Like a Human)

在支持表情反应的平台（Discord、Slack），使用 emoji 反应作为轻量级的社交信号：

```
👍 确认/赞同（不需要文字回复）
❤️ 表示欣赏
😂 觉得好笑
🤔 有趣/思考
👀 注意到了
```

**规则**：每条消息最多一个反应，不要过度使用。

### 4. 避免三连击 (Avoid Triple-Tap)

不要对同一条消息发送多个不同的反应或回复。一个深思熟虑的回复胜过三个碎片化的片段。

---

## 群组类型与策略

### 类型 1: 工作群组 (Work Groups)

**特征**：专业讨论、任务协调、项目沟通

**策略**：
```yaml
响应触发: @提及 + 工作相关问题
角色定位: 协助者/工具人
内容风格: 简洁、专业、信息优先
隐私边界: 只分享工作相关信息
```

**配置示例**：
```json5
{
  channels: {
    telegram: {
      workGroups: {
        "group_id_x": {
          responseMode: "mention-only",
          contentFilter: "professional",
          enableReactions: true
        }
      }
    }
  }
}
```

### 类型 2: 社交群组 (Social Groups)

**特征**：朋友聊天、兴趣讨论、日常分享

**策略**：
```yaml
响应触发: @提及 + 有趣话题
角色定位: 有趣的参与者/观察者
内容风格: 自然、轻松、有幽默感
隐私边界: 不分享主人私人信息
```

**配置示例**：
```json5
{
  channels: {
    telegram: {
      socialGroups: {
        "group_id_y": {
          responseMode: "smart",  // 智能判断
          contentFilter: "social",
          enableReactions: true,
          reactionFrequency: "low"  // 低频率反应
        }
      }
    }
  }
}
```

### 类型 3: 技术群组 (Technical Groups)

**特征**：技术讨论、问题解决、知识分享

**策略**：
```yaml
响应触发: @提及 + 技术问题自动识别
角色定位: 技术顾问/知识库
内容风格: 准确、有深度、带代码示例
隐私边界: 只分享技术内容
```

**配置示例**：
```json5
{
  channels: {
    discord: {
      techGroups: {
        "channel_id_z": {
          responseMode: "mention-and-keywords",
          keywords: ["bug", "error", "how to", "help"],
          contentFilter: "technical",
          enableCodeBlocks: true
        }
      }
    }
  }
}
```

---

## 配置步骤

### Telegram 群组配置

### 步骤 1: 将 Bot 添加到群组

1. 在 Telegram 搜索你的 Bot
2. 点击 Bot → Add to Group
3. 选择目标群组
4. Bot 会自动加入并获得管理员权限（可选）

### 步骤 2: 配置群组响应模式

编辑 `~/.openclaw/openclaw.json`：

```json5
{
  channels: {
    telegram: {
      botToken: "${TELEGRAM_BOT_TOKEN}",
      
      // 群组配置
      groups: {
        // 全局默认设置
        "default": {
          responseMode: "mention-only",  // 只响应 @提及
          silentMode: false,              // 不静默发送
          enableReactions: true           // 启用反应
        },
        
        // 特定群组覆盖
        "-1001234567890": {
          responseMode: "smart",          // 智能模式
          allowedTopics: ["tech", "ai"],  // 只响应特定话题
          timezone: "Asia/Shanghai"       // 时区设置
        }
      },
      
      // 访问控制
      allowFrom: ["user_id_1", "user_id_2"],  // 可选：限制用户
      adminUsers: ["admin_user_id"]            // 管理员列表
    }
  }
}
```

### 步骤 3: 配置响应触发条件

```json5
{
  channels: {
    telegram: {
      triggers: {
        // 必须触发条件
        mustMention: true,           // 必须被 @提及
        
        // 可选触发条件
        keywords: ["help", "请问", "有人知道"],  // 关键词触发
        patterns: ["怎么.*?", "为什么.*?"],      // 模式匹配
        
        // 排除条件
        excludeKeywords: ["打卡", "签到"],       // 不响应这些
        quietHours: {                          // 静默时段
          start: "23:00",
          end: "08:00",
          timezone: "Asia/Shanghai"
        }
      }
    }
  }
}
```

### Discord 群组配置

### 步骤 1: 邀请 Bot 到服务器

参考 [Discord Bot 配置最佳实践](discord-bot-configuration-best-practices.md) 的邀请流程。

### 步骤 2: 配置频道行为

```json5
{
  channels: {
    discord: {
      // 频道级别配置
      channels: {
        "general": {
          responseMode: "mention-only",
          enableThreads: false      // 不主动创建线程
        },
        
        "bot-commands": {
          responseMode: "all",       // 响应所有消息
          prefix: "/"
        },
        
        "tech-help": {
          responseMode: "smart",
          keywords: ["error", "bug", "fix"],
          autoThread: true           // 自动创建线程深入讨论
        }
      }
    }
  }
}
```

---

## 行为准则

### 消息格式准则

**Discord/WhatsApp**: 不使用 Markdown 表格
```
❌ 错误：
| 列1 | 列2 |
|-----|-----|
| A   | B   |

✅ 正确：
• 项目 1: A
• 项目 2: B
```

**Telegram**: 避免过多空行
```
❌ 错误：
这是第一段。



这是第二段。



这是第三段。

✅ 正确：
这是第一段。

这是第二段。

这是第三段。
```

**Discord 链接**: 多链接时用 `<>` 包裹避免嵌入刷屏
```
❌ 错误：
https://link1.com
https://link2.com
https://link3.com
（会生成 3 个预览卡片）

✅ 正确：
相关链接：<https://link1.com> <https://link2.com> <https://link3.com>
（无预览卡片）
```

### 私密信息处理

**核心规则**：你有权访问主人的私人信息，但这不代表你可以在群组分享它们。

```json5
// 示例场景
{
  scenario: "群里有人问主人的日程",
  
  ❌ 错误: "Andy 今天下午 3 点有牙医预约",
  
  ✅ 正确: "我不方便在群里分享 Andy 的私人日程，你可以私聊我询问"
}
```

### 群组角色认知

你是群组的一个参与者，**不是主人的代言人**：

```
❌ 错误：替主人做决定、发表观点、承诺事情
✅ 正确：提供信息、协助讨论、等待主人确认
```

---

## 消息处理最佳实践

### 响应优先级

```yaml
优先级排序:
  1. 直接 @提及 (必须响应)
  2. 明确的问题或请求 (应该响应)
  3. 能增值的技术/信息问题 (考虑响应)
  4. 一般闲聊 (保持沉默)
  5. 其他人的对话 (不介入)
```

### 响应延迟策略

```json5
{
  channels: {
    telegram: {
      responseDelay: {
        // 根据消息类型调整延迟
        mention: 0,           // @提及 立即响应
        question: 2000,       // 问题 2秒后响应（给人类机会）
        general: 5000,        // 一般消息 5秒后考虑
        
        // 智能延迟：观察是否有其他人回复
        waitForHuman: {
          enabled: true,
          timeout: 10000      // 等待 10秒，无人回复才响应
        }
      }
    }
  }
}
```

### 消息过滤

```json5
{
  channels: {
    telegram: {
      messageFilter: {
        // 不响应的消息类型
        excludeTypes: [
          "service_message",    // 系统消息（入群、改名等）
          "forwarded",          // 转发的消息
          "media_group"         // 媒体组（多张图片）
        ],
        
        // 不响应的内容模式
        excludePatterns: [
          "^\\d+$",             // 纯数字（打卡、签到）
          "^(早|晚安|晚安)[！!]", // 简单问候
          "^打卡$"              // 打卡消息
        ]
      }
    }
  }
}
```

---

## 故障排查

### 问题 1: Bot 响应过于频繁

**症状**: Bot 回复了不该回复的消息，刷屏群组

**诊断**:
```bash
# 检查响应模式配置
openclaw config get channels.telegram.groups.default.responseMode

# 查看最近的响应日志
openclaw logs | grep "group_message" | tail -20
```

**解决方案**:
```json5
{
  channels: {
    telegram: {
      groups: {
        "default": {
          responseMode: "mention-only",  // 改为严格模式
          triggers: {
            mustMention: true             // 必须被提及
          }
        }
      }
    }
  }
}
```

### 问题 2: Bot 在群组中泄露私人信息

**症状**: Bot 在群组中分享了私聊中的内容

**根因**: 缺少隐私边界配置

**解决方案**:
```json5
{
  agents: {
    defaults: {
      privacyMode: "strict",
      
      // 群组隐私规则
      groupPrivacy: {
        enabled: true,
        excludeChannels: ["telegram:group:*"],  // 排除所有群组
        sensitiveData: ["calendar", "contacts", "emails"]
      }
    }
  }
}
```

### 问题 3: Bot 不响应 @提及

**症状**: 被 @提及 后没有回复

**诊断**:
```bash
# 检查 Bot 是否在线
openclaw status | grep telegram

# 检查群组权限
curl "https://api.telegram.org/bot${TOKEN}/getChatMember?chat_id=${GROUP_ID}&user_id=${BOT_ID}"
```

**可能原因**:
1. Bot 不在群组中
2. Bot 没有读取消息权限
3. `allowFrom` 配置排除了提及者

**解决方案**:
```bash
# 确认 Bot 在群组
# 重新邀请 Bot 到群组

# 检查 allowFrom 配置
openclaw config get channels.telegram.allowFrom

# 如果设置了 allowFrom，确保提及者在列表中
openclaw config set channels.telegram.allowFrom []  # 清空限制（允许所有人）
```

### 问题 4: Bot 表情反应过于频繁

**症状**: Bot 对太多消息添加了反应

**解决方案**:
```json5
{
  channels: {
    telegram: {
      reactions: {
        enabled: true,
        mode: "minimal",        // 最小化模式
        frequency: "low",       // 低频率
        maxPerHour: 5,          // 每小时最多 5 个
        maxPerConversation: 10  // 每次对话最多 10 个
      }
    }
  }
}
```

---

## 高级配置

### 多语言群组支持

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          languages: ["zh", "en"],
          defaultLanguage: "zh",
          
          // 根据消息语言调整响应
          languageDetection: {
            enabled: true,
            respondInSameLanguage: true
          }
        }
      }
    }
  }
}
```

### 群组话题分类

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          topics: {
            // 只响应特定话题
            allowed: ["tech", "ai", "project"],
            
            // 话题识别关键词
            keywords: {
              tech: ["代码", "bug", "部署"],
              ai: ["模型", "LLM", "agent"],
              project: ["任务", "进度", "deadline"]
            }
          }
        }
      }
    }
  }
}
```

### 智能参与度控制

```json5
{
  channels: {
    telegram: {
      groups: {
        "default": {
          participationControl: {
            enabled: true,
            
            // 基于群组活跃度调整
            activityBased: {
              highActivity: "reduce",   // 群组活跃时减少参与
              lowActivity: "normal",    // 群组安静时正常参与
              threshold: 50             // 每小时消息数阈值
            },
            
            // 基于时间调整
            timeBased: {
              workHours: "professional",  // 工作时间专业风格
              offHours: "casual"          // 非工作时间轻松风格
            }
          }
        }
      }
    }
  }
}
```

### 群组数据分析

```json5
{
  monitoring: {
    groups: {
      enabled: true,
      
      metrics: [
        "response_rate",        // 响应率
        "reaction_count",       // 反应数
        "message_volume",       // 消息量
        "user_satisfaction"     // 用户满意度（基于反应）
      ],
      
      alerts: {
        highResponseRate: 0.3,  // 响应超过 30% 时警告
        lowEngagement: 0.1      // 参与度低于 10% 时提醒
      }
    }
  }
}
```

---

## 总结

### 快速检查清单

**基础配置**:
- [ ] Bot 已添加到群组
- [ ] 响应模式设置为 `mention-only` 或 `smart`
- [ ] 触发条件已配置（@提及 + 关键词）
- [ ] 隐私边界已启用

**行为准则**:
- [ ] 理解 "参与而非主导" 原则
- [ ] 知道何时说话、何时沉默
- [ ] 表情反应克制使用
- [ ] 遵守平台格式规范

**监控与调优**:
- [ ] 响应频率监控已启用
- [ ] 定期检查群组参与度
- [ ] 根据反馈调整配置

### 推荐配置模板

```json5
// ~/.openclaw/openclaw.json - 群组配置模板
{
  channels: {
    telegram: {
      botToken: "${TELEGRAM_BOT_TOKEN}",
      
      // 群组配置
      groups: {
        "default": {
          responseMode: "smart",
          triggers: {
            mustMention: false,  // 智能判断，不强制 @提及
            keywords: ["help", "请问", "@BotName"],
            excludePatterns: ["^\\d+$", "^打卡$"]
          },
          
          // 隐私保护
          privacy: {
            mode: "strict",
            excludeTopics: ["personal", "calendar"]
          },
          
          // 反应控制
          reactions: {
            enabled: true,
            mode: "minimal",
            maxPerHour: 5
          },
          
          // 响应延迟
          delay: {
            mention: 0,
            question: 3000,
            waitForHuman: {
              enabled: true,
              timeout: 10000
            }
          }
        },
        
        // 工作群组特殊配置
        "-100WORK_GROUP_ID": {
          responseMode: "mention-only",
          contentFilter: "professional",
          timezone: "Asia/Shanghai"
        }
      }
    }
  }
}
```

---

## 参考资源

- [Telegram Bot API 文档](https://core.telegram.org/bots/api)
- [Discord Bot 配置最佳实践](discord-bot-configuration-best-practices.md)
- [OpenClaw 群组配置文档](https://docs.openclaw.ai/channels/groups)
- [AI-Note 贡献指南](../docs/contribution-guide.md)

---

*本文档遵循 AI-Note 规范，欢迎提交 PR 改进。*