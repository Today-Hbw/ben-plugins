---
description: 从 PRD 到交付的完整 8 步研发流程
argument-hint: <PRD目录路径> [--skip 5,6] [--from 4] [--only 3]
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

**如果 $ARGUMENTS 为空：**
- 使用当前工作目录作为 PRD 目录
- 从步骤 1 开始执行全部 8 步

## 执行流程

### 第 0 步：环境初始化

1. 确定 PRD 目录路径：
   - 如果 $ARGUMENTS 包含路径，使用它
   - 否则使用当前工作目录

2. 扫描 PRD 目录：
   - 用 Glob 扫描所有 `.pdf` 和 `.md` 文件
   - 列出找到的文件，向用户确认

3. 解析个人标识（person）：
   - **解析顺序**：先读项目 `.claude/dev-flow.config.md`，再读全局 `~/.claude/dev-flow.config.md`，取 frontmatter 中的 `person` 字段（项目配置优先）。
   - **判定「未配置」**：两处都没有该文件，或文件存在但 `person` 字段缺失/为空。
   - **未配置时（优先询问并记录）**：
     - 用 AskUserQuestion 询问用户的个人标识（如 `ben`），说明它将用于文档署名和输出运行目录前缀。
     - 拿到后**写入全局** `~/.claude/dev-flow.config.md`：文件不存在则按 `config/default-config.md` 模板创建并填入 `person`；已存在则补充/修改 `person` 字段。
   - **已配置则直接使用**，不再询问。
   - 解析出的 `person` 即为后续所有输出文档（问答记录/计划/总结）的「负责人」署名。

4. 确定 Claude 输出目录：
   - **默认规则**：`<工作区>/Claude/<项目>/<迭代>/<person>_<时间戳>/`
     - `<工作区>`：PRD 路径中 `Prd` 目录的上级目录（如 PRD 在 `ben-workspace/Prd/ben-plugins/ben_20260630/`，则工作区为 `ben-workspace`）。
     - `<项目>/<迭代>`：`Prd/` 之后的子路径，原样镜像到 `Claude/` 下（如 `ben-plugins/ben_20260630`）。
     - `<person>_<时间戳>`：运行目录，`person` 为上一步解析出的个人标识，时间戳用 Bash `date +%Y%m%d%H%M%S`（格式 `YYYYMMDDHHMMSS`）。
     - 例：PRD 在 `ben-workspace/Prd/ben-plugins/ben_20260630/` → 输出到 `ben-workspace/Claude/ben-plugins/ben_20260630/ben_20260630171200/`
   - 若路径中无 `Prd` 段，回退为：PRD 上级目录 / `Claude` / `<person>_<时间戳>`。
   - 检查 `.claude/dev-flow.config.md` 中的 `claude_output` 等字段是否覆盖默认值。
   - 用 AskUserQuestion 向用户确认最终输出目录。

5. 创建输出目录（如果不存在）

6. 用 TodoWrite 创建 8 步任务清单，标记已完成和待执行的步骤

### 第 1 步：读 PRD

**加载 skill：** 使用 Skill 工具加载 `dev-flow:read-prd`

按 skill 指导：
- 读取所有 PRD 文件（MD 用 Read，PDF 用 Read 的 pages 参数）
- 提取每个需求的核心要点
- 识别任务拆分和 ID
- 输出需求理解摘要，向用户确认

### 第 2 步：grill me

**加载 skill：** 使用 Skill 工具加载 `dev-flow:grill`

按 skill 指导：
- 针对 PRD 中的不确定点，逐条向用户提问
- 使用 AskUserQuestion 工具提问
- 覆盖：边界条件、依赖关系、存储结构、UI 细节、异常处理
- 收集用户回答

### 第 3 步：问答记录

**加载 skill：** 使用 Skill 工具加载 `dev-flow:qa-record`

按 skill 指导：
- 将 grill 阶段的讨论整理成标准格式的 `问答记录.md`
- 写入输出目录：`<output_dir>/问答记录.md`

### 第 4 步：技术方案

**加载 skill：** 使用 Skill 工具加载 `dev-flow:tech-plan`

按 skill 指导：
- 为每个任务生成 `计划.md`
- 包含：目标、任务拆解、边界/依赖、实现要点
- 写入：`<output_dir>/<任务名_ID>/计划.md`
- 写完后向用户确认方案

### 第 5 步：编码

按技术方案实施编码：
- 逐个任务按计划编写代码
- 遵循项目现有的代码风格和规范
- 每完成一个任务，更新对应的 `计划.md` 状态为「已完成」

### 第 6 步：自测

**加载 skill：** 使用 Skill 工具加载 `dev-flow:self-test`

按 skill 指导：
- 验证代码是否符合技术方案
- 检查边界条件和异常处理
- 检查依赖和配置是否正确
- 输出自测结果

### 第 7 步：code review

**加载 skill：** 使用 Skill 工具加载 `dev-flow:code-review`

按 skill 指导：
- 审查所有代码变更
- 检查正确性、边界、性能、规范
- 输出审查意见
- 如有问题，回到第 5 步修复

### 第 8 步：总结

**加载 skill：** 使用 Skill 工具加载 `dev-flow:summary`

按 skill 指导：
- 生成开发总结 `总结.md`
- 包含：交付文件清单、上线依赖、设计要点、待联调项
- 写入：`<output_dir>/总结.md`

## 重要规则

1. **每步完成后立即保存输出文件**，不要等到最后
2. **每步完成后用 TodoWrite 更新进度**
3. **跳步时**：直接标记跳过的步骤为已完成，不执行
4. **用户中断**：如果用户要求暂停，记录当前步骤，下次运行时从该步骤继续
5. **语言**：所有输出文档使用中文
6. **确认机制**：关键步骤（步骤 2、4、7）完成后必须向用户确认再继续

## 输出文件命名规范

- 问答记录：`问答记录.md`（固定文件名）
- 计划：`<任务简称>_ID<任务编号>/计划.md`
  - 从 PRD 文件名或内容中提取任务名和 ID
  - 如：`产品配置_ID1019077/计划.md`
- 总结：`总结.md`（固定文件名）

## 错误处理

- PRD 目录不存在 → 提示用户检查路径
- PRD 目录为空（无 PDF/MD） → 提示用户确认目录
- 输出目录创建失败 → 提示权限问题
- 读取 PDF 失败 → 提示用户转换为 MD 或手动提供需求内容
