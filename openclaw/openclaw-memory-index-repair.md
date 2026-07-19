# OpenClaw Memory Search 索引问题修复最佳实践

## 问题现象

当运行 `memory_search` 时，出现以下错误：

```
memory search is paused because the memory index was built with a different embedding provider/model/settings.
index metadata is missing
Tell the user to run: openclaw memory status --index or openclaw memory index --force.
```

## 错误原因

### 根本原因

OpenClaw 的向量记忆索引（memory search）会绑定当前使用的 embedding 配置：

- **Embedding 提供者**（OpenAI / 百炼 / 火山引擎 / 本地 Ollama 等）
- **Embedding 模型名称**
- **Embedding 输出维度**
- **其他配置参数**

当这些配置发生变更后，旧的索引元数据与新配置不匹配，OpenClaw 会安全地暂停搜索功能，防止返回错误结果。

### 常见触发场景

1. **切换 embedding 提供者**：从 OpenAI 切换到百炼，再切换到火山方舟
2. **更换 embedding 模型**：从 `text-embedding-3-small` 切换到其他模型
3. **调整 embedding 配置**：修改维度、批量大小等参数
4. **系统迁移**：将 OpenClaw 迁移到新机器，索引不兼容
5. **版本升级**：OpenClaw 升级后索引格式变化

## 解决方案

### 快速修复

```bash
# 1. 先检查当前索引状态
openclaw memory status --index

# 2. 强制重建索引
openclaw memory index --force
```

### 详细步骤

1. **检查状态**：先确认索引问题
   ```bash
   openclaw memory status --index
   ```
   输出会显示当前索引状态、元数据是否匹配。

2. **强制重建**：重建整个索引
   ```bash
   openclaw memory index --force
   ```
   这会：
   - 删除旧索引
   - 使用当前配置重新计算所有记忆文件的 embedding
   - 保存新索引和元数据

3. **验证修复**：重新运行搜索测试
   ```bash
   memory_search query="test"
   ```

### 如果重建失败

如果重建过程中出错，可以尝试：

```bash
# 清理索引目录
rm -rf ~/.openclaw/data/lancedb
rm -rf ~/.openclaw/data/*.idx

# 重新构建
openclaw memory index --force
```

## 预防措施

### 最佳实践

1. **更换模型后立即重建**：修改 embedding 配置后，立刻运行 `openclaw memory index --force`，避免后续使用时出错

2. **不要频繁切换 embedding**：尽量保持稳定的 embedding 配置，减少不必要的重建

3. **备份索引**：重要部署可以备份 `~/.openclaw/data/lancedb` 目录，避免重复计算

4. **监控索引状态**：在 heartbeat 检查中定期检查索引状态

### 配置变更 Checklist

修改 OpenClaw embedding 配置时：

- [ ] 记录旧配置
- [ ] 更新配置文件
- [ ] 运行 `openclaw memory index --force`
- [ ] 测试 `memory_search` 功能正常
- [ ] 确认无错误后再继续使用

## 相关链接

- [OpenClaw 官方文档](https://docs.openclaw.ai/)
- [记忆架构设计](../memory-architecture.md)
- [OpenClaw 主页](https://github.com/openclaw/openclaw)

## 版本信息

| 日期 | 版本 | 变更 |
|------|------|------|
| 2026-07-19 | 1.0 | 初始版本，记录问题和解决方案 |

## 作者

AI Note - OpenClaw 实践沉淀
