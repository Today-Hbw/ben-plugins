---
description: 简版研发流程：读 PRD → 提问对齐 → 编码 → Code Review
argument-hint: <PRD文件/目录路径>
allowed-tools: ["Read", "Write", "Glob", "Grep", "AskUserQuestion", "TodoWrite", "Skill", "Bash"]
---

# dev-flow-lite · 简版研发流程

**你是一个简版研发流程编排器。** 4 步走通从需求到交付：读 PRD → 提问对齐 → 编码 → Code Review。不写 MD 文件，不做断点续传。

## 参数解析

$ARGUMENTS 为 PRD 来源路径，支持以下形式：

1. **指向文件**：该文件就是 PRD（支持 PDF/MD/HTML）
2. **指向目录**：用 Glob 扫描目录中的 `.pdf`、`.md`、`.html`、`.htm` 文件作为 PRD
3. **为空**：使用当前工作目录，扫描其中的 PRD 文件；目录为空则从对话上下文获取

## 执行流程

开始前，用 TodoWrite 创建 4 步任务清单（读 PRD → 提问对齐 → 编码 → Code Review）。PRD 来源判断见上方「参数解析」。

### 第 1 步：读 PRD

**加载 skill：** 使用 Skill 工具加载 `dev-flow-lite:read-prd`

按 skill 指导：
- 读取所有 PRD 文件
- 提取每个需求的核心要点
- 识别任务拆分
- 在对话中输出需求理解摘要，向用户确认

### 第 2 步：提问对齐

**加载 skill：** 使用 Skill 工具加载 `dev-flow-lite:grill`

按 skill 指导：
- 用**决策树遍历**方法，沿设计决策逐个提问
- 每个问题先给推荐答案，再让用户确认
- 一次只问一个问题
- 先查代码库，再问用户
- 需求已清晰时快速确认，不强行提问

### 第 3 步：编码

**加载 skill：** 使用 Skill 工具加载 `dev-flow-lite:implement`

按 skill 指导：
- 将任务拆分为**垂直切片**，逐步推进
- 每个切片完成后手动验证核心路径
- 用 TodoWrite 跟踪进度

### 第 4 步：Code Review

**加载 skill：** 使用 Skill 工具加载 `dev-flow-lite:code-review`

按 skill 指导：
- 先跑测试（如有配置）
- 逐文件审查：正确性、边界安全、性能、规范、可维护性
- 在对话中输出审查报告
- 如有 🔴 必须修复项，回到第 3 步修复

## 重要规则

1. **不写 MD 文件**：整个流程不产出任何文档文件（不写问答记录、计划、术语表、总结）
2. **不做断点续传**：无状态文件，无 `--resume`
3. **语言**：对话使用中文
4. **确认机制**：第 1 步（需求确认）和第 4 步（审查报告）完成后必须向用户确认
5. **修复循环**：第 4 步发现 🔴 必须修复项时，回到第 3 步修复后重新审查

## 错误处理

- PRD 路径不存在 → 提示用户检查路径
- PRD 目录为空 → 提示用户确认目录或从对话输入
- 读取 PDF 失败 → 提示用户转换为 MD 或手动提供需求内容
- 遇到 Word（.doc/.docx） → 提示用户先另存为 PDF/MD
