# dev-flow 插件

个人研发流程插件，定义从需求到交付的完整 8 步工作流。

## 核心理念

让 Claude 在每个项目中自动遵循你的研发流程，输出标准格式的文档。

## 8 步工作流

| 步骤 | 名称 | 输出 |
|------|------|------|
| 1 | 读 PRD | 需求理解摘要 |
| 2 | grill me | 向用户提问 |
| 3 | 问答记录 | `问答记录.md` |
| 4 | 技术方案 | `任务名_ID/计划.md` |
| 5 | 编码 | 代码文件 |
| 6 | 自测 | 测试报告 |
| 7 | code review | 审查意见 |
| 8 | 总结 | `总结.md` |

## 使用方式

### 传入 PRD 目录路径

```bash
/dev-flow D:/path/to/ben_prd/项目名/迭代名/人名
```

### 当前目录就是 PRD

```bash
cd D:/path/to/ben_prd/项目名/迭代名/人名
/dev-flow
```

### 跳步控制

```bash
/dev-flow --skip 5,6    # 跳过编码和自测
/dev-flow --from 4      # 从步骤 4 开始
/dev-flow --only 3      # 只做步骤 3（问答记录）
```

## 输出目录结构

输出目录把 PRD 的 `Prd/<项目>/<迭代>` 子树镜像到工作区的 `Claude/` 下，并加一层 `<个人标识>_<时间戳>` 运行目录：

**规则**：`<工作区>/Claude/<项目>/<迭代>/<person>_<YYYYMMDDHHMMSS>/`
（工作区 = PRD 路径中 `Prd` 目录的上级目录）

```
ben-workspace/
├── Prd/
│   └── ben-plugins/            ← 项目
│       └── ben_20260630/       ← 迭代
│           ├── 需求1.pdf       ← 你的 PRD（PDF/MD）
│           └── 需求2.md
└── Claude/
    └── ben-plugins/
        └── ben_20260630/
            └── ben_20260630171200/   ← <个人标识>_<时间戳> 运行目录
                ├── 问答记录.md
                ├── 总结.md
                ├── 需求1_ID001/
                │   └── 计划.md
                └── 需求2_ID002/
                    └── 计划.md
```

> 个人标识（`person`）来自配置文件；未配置时 dev-flow 会在第 0 步优先询问并写入全局 `~/.claude/dev-flow.config.md`。

## 配置文件

可放在项目 `.claude/dev-flow.config.md`（仅当前项目）或全局 `~/.claude/dev-flow.config.md`（跨项目复用），**项目配置优先**：

```markdown
---
claude_output: Claude
person: ben          # 个人标识：用于文档署名 + 运行目录前缀
language: zh-CN
---

## 个人偏好
（自定义编码习惯、命名规范等）
```

> **个人标识（person）**：若两处配置都没有该字段（或为空），dev-flow 在第 0 步会**优先询问**并把结果**写入全局** `~/.claude/dev-flow.config.md`，之后不再重复询问。

## 插件结构

```
dev-flow/
├── commands/dev-flow.md          # 主命令（8 步编排）
├── skills/
│   ├── read-prd/SKILL.md        # 读 PRD 方法
│   ├── grill/SKILL.md           # 提问策略
│   ├── qa-record/SKILL.md       # 问答记录格式
│   ├── tech-plan/SKILL.md       # 技术方案格式
│   ├── self-test/SKILL.md       # 自测方法
│   ├── code-review/SKILL.md     # CR 检查项
│   └── summary/SKILL.md         # 总结格式
└── config/default-config.md      # 配置模板
```

## 自定义扩展

每个阶段的输出格式和方法论都通过 skill 文件定义，你可以：
- 修改现有 skill 调整输出格式
- 添加新的 skill 覆盖默认行为
- 在配置文件中添加个人偏好

## License

MIT
