---
description: 从 PRD 到交付的完整 8 步研发流程
argument-hint: <PRD目录路径> [--skip 5,6] [--from 4] [--only 3] [--resume]
allowed-tools: ["Read", "Write", "Glob", "Grep", "AskUserQuestion", "TodoWrite", "Skill", "Bash"]
---

# dev-flow · 研发流程编排

**你是一个研发流程编排器。** 根据用户的指令，按 8 步工作流推进，每步加载对应 skill 执行。

## 参数解析

$ARGUMENTS 可能包含：

1. **PRD 目录路径**（必须）：PRD 文件所在的目录（PDF/MD 文件）
2. **流程控制参数**（可选）：
   - `--skip N,M`：跳过指定步骤（如 `--skip 5,6` 跳过编码和自测）
   - `--from N`：从第 N 步开始（如 `--from 4` 从技术方案开始）
   - `--only N`：只执行第 N 步
   - `--resume`：从上次中断的步骤继续（读取输出目录的 `.dev-flow-state.json`）

**如果 $ARGUMENTS 为空：**
- 使用当前工作目录作为 PRD 目录
- 从步骤 1 开始执行全部 8 步

## 会话恢复

状态文件路径：`<session_dir>/.dev-flow-state.json`

状态文件格式：
```json
{
  "prd_dir": "D:/path/to/prd_root/项目名/person_YYYYMMDD",
  "session_dir": "D:/path/to/prd_root/Claude/项目名/person_YYYYMMDD/person__20260630153000",
  "current_step": 4,
  "completed_steps": [1, 2, 3],
  "generated_files": {
    "qa_record": "问答记录.md",
    "glossary": "术语表.md",
    "plans": ["任务1_ID001/计划.md", "任务2_ID002/计划.md"]
  },
  "last_updated": "2026-06-30T10:30:00Z"
}
```

**恢复逻辑：**
1. `--resume` 参数触发时，读取当前批次目录（`<person>_YYYYMMDD`）下最新的会话目录的 `.dev-flow-state.json`
2. 向用户确认："检测到上次进度到步骤 N（已完成步骤 X, Y, Z），是否继续？"
3. 用户确认后，从 `current_step` 开始执行
4. 每步完成后立即更新状态文件

## 执行流程

### 第 0 步：环境初始化

1. **检查恢复状态**（最先执行）：
   - 如果传入了 `--resume` 参数，查找最近的 Claude 输出目录中的会话目录
   - 如果有会话目录包含 `.dev-flow-state.json`：
     - 读取状态文件，向用户确认是否恢复
     - 用户确认恢复 → 跳到 `current_step` 继续执行
     - 用户选择重新开始 → 删除状态文件，创建新的会话目录，从步骤 1 开始

2. **确定 PRD 来源**（三种情况）：

   **情况 A：路径指向文件**
   - 如果 $ARGUMENTS 指向一个 PDF/MD 文件，该文件就是 PRD
   - PRD 目录 = 该文件所在目录

   **情况 B：路径指向目录**
   - 如果 $ARGUMENTS 指向一个目录，或当前工作目录
   - 用 Glob 扫描目录中的所有 `.pdf` 和 `.md` 文件
   - 如果找到文件 → PRD 就是这些文件

   **情况 C：没有文件（对话输入）**
   - 如果目录为空，或用户明确说 PRD 是对话内容
   - 从对话上下文获取 PRD 内容
   - 用 AskUserQuestion 让用户确认/补充 PRD 内容

3. **解析个人标识（person）**：

   后续批次目录、会话目录的前缀和文档署名都用到 `person`，须先确定它。

   - **解析顺序**：先读项目（PRD 根目录）`.claude/dev-flow.config.md`，再读全局 `~/.claude/dev-flow.config.md`，取 frontmatter 中的 `person` 字段（项目配置优先）。
   - **判定「未配置」**：两处都没有该文件，或文件存在但 `person` 字段缺失/为空。
   - **未配置时（优先询问并记录）**：
     - 用 AskUserQuestion 询问用户的个人标识（如 `ben`），说明它将用于批次/会话目录前缀和文档署名。
     - 拿到后**写入全局** `~/.claude/dev-flow.config.md`：文件不存在则按 `config/default-config.md` 模板创建并填入 `person`；已存在则补充/修改 `person` 字段。
   - **已配置则直接使用**，不再询问。
   - 若 PRD 路径中的批次目录名（如 `ben_20260630`）与配置的 `person` 不一致，以配置的 `person` 为准，并提示用户。
   - 解析出的 `person` 即为后续所有输出文档（问答记录/计划/总结）的「负责人」署名。

4. **确定批次目录**：

   批次目录用于归集同一批次的多次会话，命名格式为 `<person>_YYYYMMDD` 或 `<person>_YYYYMMDD_N`（同一天第 N 批）。

   **批次目录由用户手动创建**，代表一次独立的需求分配。dev-flow 只负责从路径中识别批次目录名，并在 Claude/ 下镜像创建同名目录。

   **检测逻辑：**

   1. **从 PRD 路径提取**（最常用）：
      - 如果 PRD 路径中包含 `ben_YYYYMMDD` 或 `ben_YYYYMMDD_N` 格式的目录名，直接用它作为批次目录名
      - 例：`Prd/亲亲创客合伙人二期/ben_20260630_2/` → 批次目录名 = `ben_20260630_2`
      - Claude 输出镜像创建同名批次目录

   2. **路径中不含批次目录**：
      - 提示用户：请先在 Prd 下创建批次目录（如 `ben_20260630/`）并写好 `分配.md`，再从该目录运行 dev-flow
      - 或用 AskUserQuestion 让用户指定批次目录名

   **同一天多批次示例：**
   ```
   Prd/项目名/
   ├── ben_20260630/      ← 第一批：手动创建 + 写分配.md + 跑 dev-flow
   ├── ben_20260630_2/    ← 第二批：同一天又分到新需求，再建一个
   └── ben_20260630_3/    ← 第三批
   ```

5. **确定输出目录**：

   **情况 A：有项目结构**（路径形如 `root/项目名/person_YYYYMMDD/`）
   - 输出到 `root/Claude/项目名/person_YYYYMMDD/`
   - 例：`ben-workspace/亲亲创客合伙人二期/ben_20260630/` → `ben-workspace/Claude/亲亲创客合伙人二期/ben_20260630/`

   **情况 B：没有项目结构**（普通目录或文件）
   - 输出到 `<PRD目录>/../Claude/<person>_YYYYMMDD/`
   - 例：`D:/work/prd/` → `D:/work/Claude/ben_20260630/`

   **情况 C：对话输入**（无文件）
   - 输出到 `./output/Claude/<person>_YYYYMMDD/`（当前工作目录）

   **配置文件优先**：
   - 检查是否有 `.claude/dev-flow.config.md` 配置文件
   - 如果配置了 `claude_output` 绝对路径，直接使用
   - 否则按上述规则推断
   - 用 AskUserQuestion 向用户确认输出目录

6. **创建会话目录**：
   - 在批次目录下创建 `<person>__<YYYYMMDDHHmmss>` 格式的会话目录
   - 例：`ben_20260630/ben__20260630153000`
   - 所有流程产出的文档都写在这个会话目录里
   - 每次运行新建一个会话目录，不会覆盖之前的输出

7. 创建输出目录（如果不存在）

8. 用 TodoWrite 创建 8 步任务清单，标记已完成和待执行的步骤

9. **初始化状态文件**：写入 `<会话目录>/.dev-flow-state.json`，`current_step: 1`

### 第 1 步：读 PRD

**加载 skill：** 使用 Skill 工具加载 `dev-flow:read-prd`

按 skill 指导：
- 读取所有 PRD 文件（MD 用 Read，PDF 用 Read 的 pages 参数）
- 提取每个需求的核心要点
- 识别任务拆分和 ID
- 输出需求理解摘要，向用户确认

完成后更新 `.dev-flow-state.json`。

### 第 2 步：grill me

**加载 skill：** 使用 Skill 工具加载 `dev-flow:grill`

按 grill skill 指导：
- 用**决策树遍历**方法，沿设计决策向下走，逐个解决分支间依赖
- 每个问题先给推荐答案，再让用户确认
- 一次只问一个问题
- 先查代码库，再问用户
- **同步构建术语表**：发现术语歧义时立即追问并记录，grill 完成后输出 `<session_dir>/术语表.md`

完成后更新 `.dev-flow-state.json`。

### 第 3 步：问答记录

**加载 skill：** 使用 Skill 工具加载 `dev-flow:qa-record`

按 skill 指导：
- 将 grill 阶段的所有提问和回答整理成结构化的 `问答记录.md`
- 写入输出目录：`<session_dir>/问答记录.md`

完成后更新 `.dev-flow-state.json`。

### 第 4 步：技术方案

**加载 skill：** 使用 Skill 工具加载 `dev-flow:tech-plan`

按 skill 指导：
- 为每个任务生成 `计划.md`，采用**行为化描述**风格
- 包含：目标、验收标准、任务拆解、边界/依赖（含作用域外）
- 写入：`<session_dir>/<任务名_ID>/计划.md`
- 写完后用 AskUserQuestion 向用户确认方案

完成后更新 `.dev-flow-state.json`。

### 第 5 步：编码

**加载 skill：** 使用 Skill 工具加载 `dev-flow:implement`

按 implement skill 指导：
- 将技术方案拆分为**垂直切片**（每个切片走通全链路：数据→接口→展示→测试）
- 有测试框架时跑 **TDD 循环**：RED → GREEN → REFACTOR
- 测试运行节奏：每次提交前跑类型检查，每个切片完成后跑单文件测试，全部完成后跑完整套件
- 每完成一个切片，更新 `计划.md` 中对应子任务状态
- 遵循项目现有的代码风格和规范（参考 dev-flow 配置文件中的个人偏好）

完成后更新 `.dev-flow-state.json`。

### 第 6 步：自测

**加载 skill：** 使用 Skill 工具加载 `dev-flow:self-test`

按 skill 指导：
- 先检查**测试质量**（测试是否通过公共接口验证行为，是否能扛住重构）
- 逐条对照计划.md 的验收标准检查是否满足
- 检查边界条件、数据一致性、依赖、代码质量、兼容性
- 输出自测结果

完成后更新 `.dev-flow-state.json`。

### 第 7 步：code review

**加载 skill：** 使用 Skill 工具加载 `dev-flow:code-review`

按 skill 指导：
- **审查前验证**（先跑代码）：
  - 自动检测项目测试配置（package.json 的 test script 等）
  - 有测试则先跑，将结果纳入审查报告
  - 无测试则跳过，标注"仅做静态审查"
- 逐文件审查：正确性、边界安全、性能、规范、可维护性
- 输出审查报告
- 如有 🔴 必须修复项，回到第 5 步修复

完成后更新 `.dev-flow-state.json`。

### 第 8 步：总结

**加载 skill：** 使用 Skill 工具加载 `dev-flow:summary`

按 skill 指导：
- 生成开发总结 `总结.md`
- 包含：交付文件清单、上线依赖、设计要点、待联调项
- 写入：`<session_dir>/总结.md`
- 更新每个任务的 `计划.md` 状态为「已完成（详见 ../总结.md）」

完成后删除 `.dev-flow-state.json`（流程已结束）。

## 重要规则

1. **每步完成后立即保存输出文件**，不要等到最后
2. **每步完成后更新 `.dev-flow-state.json`**，记录当前进度
3. **每步完成后用 TodoWrite 更新进度**
4. **跳步时**：直接标记跳过的步骤为已完成，不执行
5. **用户中断**：状态文件已保存，下次用 `--resume` 可继续
6. **语言**：所有输出文档使用中文
7. **确认机制**：关键步骤（步骤 2、4、7）完成后必须向用户确认再继续

## 输出目录结构

### 场景 A：有项目结构

```
ben-workspace/
├── Prd/
│   └── 亲亲创客合伙人二期/              ← PRD 项目
│       ├── all/                        ← 所有人共用的 PRD（唯一数据源）
│       ├── ben_20260629/               ← ben 第一批任务
│       │   └── 分配.md                 ← 索引文件，指向 all/ 中的 PRD
│       └── ben_20260630/               ← ben 第二批任务
│           └── 分配.md
│
└── Claude/
    └── 亲亲创客合伙人二期/              ← 镜像 PRD 的项目层级
        ├── ben_20260629/
        │   └── ben__20260629183000/    ← 会话目录
        │       ├── .dev-flow-state.json
        │       ├── 术语表.md
        │       ├── 问答记录.md
        │       ├── 任务名_ID001/计划.md
        │       └── 总结.md
        └── ben_20260630/
            └── ben__20260630153000/    ← 同一天的新会话
                └── ...
```

### 场景 B：没有项目结构（普通目录）

```
D:/work/my-project/
├── prd/                                ← PRD 目录（或单个文件）
│   └── 需求.md
│
└── Claude/
    └── ben_20260630/                   ← 批次目录（按当天日期）
        └── ben__20260630153000/        ← 会话目录
            └── ...
```

### 场景 C：无文件（对话输入）

```
当前工作目录/
└── output/
    └── Claude/
        └── ben_20260630/               ← 批次目录
            └── ben__20260630153000/    ← 会话目录
                └── ...
```

## 输出文件命名规范

- 批次目录：`<person>_YYYYMMDD`（如 `ben_20260630`，按当天日期自动创建）
- 会话目录：`<person>__<YYYYMMDDHHmmss>`（时间戳精确到秒，在批次目录下）
- 问答记录：`问答记录.md`（固定文件名）
- 术语表：`术语表.md`（固定文件名）
- 计划：`<任务简称_ID<任务编号>/计划.md`
  - 从 PRD 文件名或内容中提取任务名和 ID
  - 如：`产品配置_ID1019077/计划.md`
- 总结：`总结.md`（固定文件名）
- 状态文件：`.dev-flow-state.json`（流程结束后删除）

## 错误处理

- PRD 目录不存在 → 提示用户检查路径
- PRD 目录为空（无 PDF/MD） → 提示用户确认目录
- 输出目录创建失败 → 提示权限问题
- 读取 PDF 失败 → 提示用户转换为 MD 或手动提供需求内容
- 状态文件损坏 → 提示用户删除状态文件重新开始
