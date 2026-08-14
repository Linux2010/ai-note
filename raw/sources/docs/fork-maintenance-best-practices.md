# Fork 仓库维护最佳实践指南

## 问题背景

在开源贡献过程中，经常遇到 PR 被拒绝的情况，其中一个重要原因是 **fork 仓库的 main 分支不干净**。这通常表现为：

- 包含大量本地工作文件（如 `agents/` 目录、会话记录、临时文件等）
- 与官方仓库的提交历史不一致
- 包含与当前 PR 无关的更改

## 核心原则

### 1. Main 分支作为保护分支
个人 fork 的 `main` 分支必须始终保持与官方仓库完全一致，**绝不**在上面进行任何开发或提交。

### 2. 特性分支开发模式
所有开发工作必须在基于 `main` 创建的特性分支上进行：
```bash
git checkout main
git checkout -b fix/issue-description
```

### 3. 定期同步官方仓库
每次贡献前都要确保基于最新官方代码：
```bash
git checkout main
git pull upstream main
```

## 问题诊断与修复

### 诊断步骤
1. 检查本地 main 分支与官方仓库是否一致：
   ```bash
   git log --oneline -5 origin/main
   git log --oneline -5 upstream/main
   ```
2. 查看差异文件：
   ```bash
   git diff origin/main upstream/main --name-only
   ```

### 修复流程
当发现 fork 仓库被污染时，按以下步骤修复：

#### 步骤 1: 强制重置 main 分支
```bash
git checkout main
git fetch upstream
git reset --hard upstream/main
```

#### 步骤 2: 清理本地污染文件
```bash
git clean -fdx  # 删除所有未跟踪文件和目录
```

#### 步骤 3: 强制推送纯净状态
```bash
git push origin main --force
```

#### 步骤 4: 重新创建特性分支
```bash
git checkout -b fix/clean-implementation
# 进行开发工作
```

## 预防措施

### 1. 配置 .gitignore
确保工作区文件不会意外提交：
```
# OpenClaw 工作区文件
agents/
sessions/
workspace*
*.jsonl
*.deleted.*
*.reset.*
```

### 2. 自动化同步脚本
创建同步脚本 `sync-fork.sh`：
```bash
#!/bin/bash
git checkout main
git pull upstream main
git push origin main
echo "Fork synchronized with upstream"
```

### 3. PR 提交前检查清单
- [ ] 分支是否基于最新官方 main 创建？
- [ ] 是否只包含相关更改？
- [ ] 是否有意外的文件被添加？
- [ ] 本地 main 分支是否干净？

## 常见错误场景

### 场景 1: 本地文件污染
**症状**: PR 包含 `agents/coding/workspace/` 等无关文件  
**原因**: 在 main 分支上直接运行了 OpenClaw  
**解决方案**: 使用 `git clean -fdx` 清理并重置

### 场景 2: 提交历史混乱
**症状**: 本地提交哈希与官方不一致  
**原因**: 在 main 分支上进行了开发提交  
**解决方案**: 强制重置到官方状态

### 场景 3: 同步不及时
**症状**: PR 与官方代码冲突  
**原因**: 长时间未同步官方更新  
**解决方案**: 定期执行同步脚本

## 自动化工具支持

OpenClaw 的 `github-contribution` 技能已经集成了这些最佳实践：

- 自动配置 upstream 远程
- 自动同步最新代码
- 自动创建规范的特性分支
- 避免在 main 分支上直接操作

使用示例：
```bash
./github-contribution.sh username openclaw/openclaw 123 fix-description
```

## 总结

维护一个干净的 fork 仓库是成功开源贡献的基础。通过遵循 **保护 main 分支、特性分支开发、定期同步** 的原则，可以避免大多数 PR 被拒绝的问题。记住：**你的 fork 的 main 分支应该永远是官方仓库的精确镜像**。

---
**最后更新**: 2026-03-03  
**相关记忆**: `/Users/hope/IdeaProjects/openclaw/MEMORY.md` (搜索 "Fork 仓库维护陷阱")