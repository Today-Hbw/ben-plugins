# dev-flow 插件

个人研发流程插件，定义从需求到交付的完整 8 步工作流。

## 核心理念

让 Claude 在每个项目中自动遵循你的研发流程，输出标准格式的文档。

## 8 步工作流

| 步骤 | 名称 | 输出 | 关键方法 |
|------|------|------|---------|
| 1 | 读 PRD | 需求理解摘要 | 文件扫描 + 信息提取 |
| 2 | grill me | 向用户提问 + 术语表 | **决策树遍历** + 领域建模 |
| 3 | 问答记录 | `问答记录.md` | 结构化 Q/A 输出 |
| 4 | 技术方案 | `任务名_ID/计划.md` | **Agent Brief 风格**（行为化描述） |
| 5 | 编码 | 代码文件 | **垂直切片** + TDD 循环 |
| 6 | 自测 | 测试报告 | 行为测试验证 + 质量检查 |
| 7 | code review | 审查意见 | **验证前置**（先跑测试再审查） |
| 8 | 总结 | `总结.md` | 交付物记录 |

## 使用方式

### 传入 PRD 目录路径

```bash
/dev-flow D:/path/to/ben-workspace/项目名/迭代名/人名_YYYYMMDD
```

### 当前目录就是 PRD

```bash
cd D:/path/to/ben-workspace/项目名/迭代名/人名_YYYYMMDD
/dev-flow
```

### 跳步控制

```bash
/dev-flow --skip 5,6    # 跳过编码和自测
/dev-flow --from 4      # 从步骤 4 开始
/dev-flow --only 3      # 只做步骤 3（问答记录）
/dev-flow --resume      # 从上次中断处继续
/dev-flow --overview    # 维护工作区 总览.md 进度索引（默认不碰）
```

`--overview` 会在需求确认后登记到 `总览.md` 的「在办」，交付后挪到「已完成」；不传则完全不碰 `总览.md`。需 PRD 位于 `Prd/` 工作区结构下才能定位。

## 输出目录结构

Claude 输出目录根据 PRD 的位置自动创建。批次目录按当天日期自动归集，每次运行新建一个带时间戳的会话目录：

### 有项目结构时

```
ben-workspace/
├── Prd/
│   └── 亲亲创客合伙人二期/              ← PRD 项目
│       ├── all/                        ← 公共 PRD（唯一数据源）
│       └── ben_20260630/               ← 你的批次目录
│           └── 分配.md                 ← 索引文件
│
└── Claude/
    └── 亲亲创客合伙人二期/              ← 镜像 PRD 的项目层级
        └── ben_20260630/               ← 批次目录（按当天日期）
            └── ben__20260630153000/    ← 会话目录
                ├── 问答记录.md
                ├── 任务名_ID001/计划.md
                └── 总结.md
```

### 没有项目结构时

PRD 只是一个普通目录或文件：

```
D:/work/my-project/
├── prd/需求.md                             ← PRD 文件
└── Claude/ben_20260630/ben__20260630153000/ ← 输出
```

### 对话输入（无文件）

PRD 是对话中的一句话，输出在当前工作目录：

```
./output/Claude/ben_20260630/ben__20260630153000/  ← 输出
```

> 个人标识（`person`）来自配置文件；未配置时 dev-flow 会在第 0 步优先询问并写入全局 `~/.claude/dev-flow.config.md`。

## 配置文件

可放在 PRD 根目录的 `.claude/dev-flow.config.md`（仅当前项目）或全局 `~/.claude/dev-flow.config.md`（跨项目复用），**项目配置优先**：

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
├── commands/dev-flow.md          # 主命令（8 步编排 + 会话恢复）
├── skills/
│   ├── read-prd/SKILL.md        # 读 PRD 方法
│   ├── grill/SKILL.md           # 提问策略（决策树遍历 + 术语表构建 v2）
│   ├── qa-record/SKILL.md       # 问答记录格式
│   ├── tech-plan/SKILL.md       # 技术方案格式（Agent Brief 风格 v2）
│   ├── implement/SKILL.md       # 编码实现（垂直切片 + TDD）🆕
│   ├── self-test/SKILL.md       # 自测方法（含测试哲学 v2）
│   ├── code-review/SKILL.md     # CR 检查项（含验证前置 v2）
│   └── summary/SKILL.md         # 总结格式
└── config/default-config.md      # 配置模板
```

🆕 = 本次新增的 skill

## 自定义扩展

每个阶段的输出格式和方法论都通过 skill 文件定义，你可以：
- 修改现有 skill 调整输出格式
- 添加新的 skill 覆盖默认行为
- 在配置文件中添加个人偏好

## 参考项目

本插件的设计参考了以下优秀项目：

- [mattpocock/skills](https://github.com/mattpocock/skills) - Matt Pocock 的 Claude Code Skills 最佳实践
- [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official) - Anthropic 官方 Claude 插件示例

## License

MIT
