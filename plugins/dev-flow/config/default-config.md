# dev-flow 配置模板

将以下内容复制到项目的 `.claude/dev-flow.config.md` 文件中，按需修改。

```markdown
---
# Claude 输出目录（相对于 PRD 上级目录）
claude_output: Claude

# 你的标识（用于输出子目录名）
person: ben

# 文档语言
language: zh-CN

# 默认起始步骤（1-8）
default_from: 1

# 默认跳过的步骤（逗号分隔，如 "6,7"）
default_skip: ""
---

## 个人偏好

### 编码习惯
<!-- 在此补充你的编码习惯，Claude 在步骤 5 编码时会参考 -->

- 变量命名：camelCase
- 函数命名：camelCase
- 类命名：PascalCase
- 文件命名：kebab-case
- 注释语言：中文

### 代码风格
<!-- 在此补充你偏好的代码风格 -->

- 使用 async/await 而非 Promise.then
- 错误处理使用 try/catch
- 日志使用项目统一 logger

### 沟通偏好
<!-- 在此补充你和 Claude 交互时的偏好 -->

- 回复语言：中文
- 回复风格：简洁直接
- 不确定时先问我，不要自己猜
```

## 说明

- 配置文件可放在两处：项目的 `.claude/dev-flow.config.md`（仅当前项目生效）或全局 `~/.claude/dev-flow.config.md`（跨项目复用）。**项目配置优先于全局配置。**
- 如果两处都没有，插件使用默认值。
- `person`（个人标识）字段有两个用途：
  1. **文档署名**：问答记录 / 计划 / 总结 文档中的「负责人」。
  2. **运行目录前缀**：输出目录的运行子目录命名为 `<person>_<时间戳>`，如 `ben_20260630171200`。
- **未配置时优先询问并记录**：若两处配置都没有 `person`（或为空），dev-flow 在第 0 步会**优先询问**你的个人标识，并把结果**写入全局** `~/.claude/dev-flow.config.md`，之后跨项目复用、不再重复询问。
- 输出目录默认规则：`<工作区>/Claude/<项目>/<迭代>/<person>_<时间戳>/`，其中工作区为 PRD 路径中 `Prd` 目录的上级目录。
- 个人偏好部分可以随意扩展，Claude 在对应步骤会参考这些偏好
