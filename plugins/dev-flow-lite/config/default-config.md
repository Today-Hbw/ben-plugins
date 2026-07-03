# dev-flow-lite 配置模板

dev-flow-lite 与 dev-flow 共用同一个配置文件。将以下内容复制到 `~/.claude/dev-flow.config.md`（全局）或项目的 `.claude/dev-flow.config.md`（仅当前项目）。**项目配置优先。**

> 注意：如果你同时使用完整版 dev-flow，配置中的 `person` 字段由完整版管理，lite 会忽略它。

```markdown
---
# 你的标识（仅完整版 dev-flow 使用，lite 忽略）
person: ben

# 文档语言
language: zh-CN
---

## 个人偏好

### 沟通偏好

- 回复语言：中文
- 回复风格：简洁直接
- 不确定时先问我，不要自己猜
```

## 说明

- dev-flow-lite 只读取 `## 个人偏好` 部分，用于编码时参考（命名规范、沟通风格等）
- `person`、`language` 等 frontmatter 字段在 lite 中被忽略
- 没有配置文件时 lite 正常工作，使用默认行为
