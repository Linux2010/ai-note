# AI-Note 索引

## 项目结构

```
ai-note/
├── README.md                 # 项目介绍
├── index.md                  # 本索引文件
└── [domain]/                # 按领域分类的文档
    ├── [topic].md           # 具体技术文档
    └── ...
```

## 文档分类

### OpenClaw
- **架构设计**
  - [多代理架构完整方案](openclaw/architecture/multi-agent.md)
- **技术解决方案**  
  - [代理问题解决方案](openclaw/solutions/proxy-solutions.md)

### 其他领域
- *(待添加)*

## 文档规范

所有文档遵循以下元数据结构：
```yaml
文档类型: [架构设计|技术解决方案|最佳实践]
适用场景: [具体使用场景]
最后更新: YYYY-MM-DD
```

## 贡献流程

1. 在对应领域目录下创建文档
2. 更新本索引文件
3. 提交 PR 到主仓库
4. 确保文档 AI 友好（结构化、可解析）

---
**索引最后更新**: 2026-02-27