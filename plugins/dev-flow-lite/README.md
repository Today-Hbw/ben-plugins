# dev-flow-lite 插件

简版研发流程插件，4 步走通从需求到交付。

## 核心理念

**极简**：只保留必要步骤，不写 MD 文件，不做断点续传。适合小需求、快速迭代。

## 与 dev-flow 的区别

| | dev-flow（完整版） | dev-flow-lite（简版） |
|---|---|---|
| 步骤 | 8 步 | 4 步 |
| 文档输出 | 问答记录、计划、术语表、总结 | 无 |
| 断点续传 | ✅ `--resume` | ❌ |
| 路径判断 | 快速/标准/文档 | ❌ |
| 适用场景 | 复杂需求、跨模块 | 小需求、快速迭代 |

## 4 步工作流

| 步骤 | 名称 | 做什么 |
|------|------|--------|
| 1 | 读 PRD | 扫描 PRD 文件，提取要点，对话中确认需求 |
| 2 | 提问对齐 | 决策树遍历，逐个提问消除歧义 |
| 3 | 编码实现 | 按切片逐步推进，手动验证 |
| 4 | Code Review | 先跑测试，再逐文件审查 |

## 使用方式

### 传入 PRD 路径

```bash
/dev-flow-lite D:/path/to/prd/需求.md
```

### 当前目录就是 PRD

```bash
cd D:/path/to/prd/
/dev-flow-lite
```

## 配置文件

与完整版 dev-flow 共用同一个配置文件，放在 `~/.claude/dev-flow.config.md`（全局）或项目的 `.claude/dev-flow.config.md`（项目优先）。lite 只读取 `## 个人偏好` 部分（编码风格、沟通偏好等），忽略 `person` 等 frontmatter 字段。

```markdown
---
person: ben          # 仅完整版使用，lite 忽略
language: zh-CN
---

## 个人偏好
（自定义编码习惯、命名规范等）
```

## 插件结构

```
dev-flow-lite/
├── commands/dev-flow-lite.md       # 主命令（4 步编排）
├── skills/
│   ├── read-prd/SKILL.md          # 读 PRD 方法
│   ├── grill/SKILL.md             # 提问策略（决策树遍历）
│   ├── implement/SKILL.md         # 编码实现（垂直切片）
│   └── code-review/SKILL.md       # CR 检查项（验证前置）
└── config/default-config.md       # 配置模板
```

## License

MIT
