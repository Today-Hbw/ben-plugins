# Claude Code Plugins - ben

ben 的 Claude Code 插件集合，包含个人研发流程等自定义插件。

## 结构

- **`/plugins`** - 内部插件

## 安装

在 Claude Code 中运行：

```
/plugin marketplace add Today-Hbw/ben-plugins
/plugin install dev-flow@ben-plugins
/plugin install dev-flow-lite@ben-plugins


# 指定版本
/plugin marketplace add https://github.com/Today-Hbw/ben-plugins.git#v1.0.0
# 指定分支
/plugin marketplace add https://github.com/Today-Hbw/ben-plugins.git#your-branch-name
```

或在 `/plugin > Discover` 中浏览。

## 插件列表

| 插件 | 说明 |
|------|------|
| [dev-flow](plugins/dev-flow) | 从 PRD 到交付的完整 8 步研发工作流 |
| [dev-flow-lite](plugins/dev-flow-lite) | 简版研发流程：读 PRD → 提问 → 编码 → CR（不写 MD、不断点续传） |
| [uniplat](plugins/uniplat) | Uniplat 低代码平台开发辅助插件 |

## 插件结构

每个插件遵循标准结构：

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json      # 插件元数据（必需）
├── commands/            # 斜杠命令（可选）
├── skills/              # 技能定义（可选）
├── config/              # 配置模板（可选）
└── README.md            # 文档
```

## 参考项目

本仓库的插件设计参考了以下项目：

- [mattpocock/skills](https://github.com/mattpocock/skills) — Matt Pocock 的 Claude Code Skills 集合
- [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official) — Anthropic 官方 Claude 插件仓库

## License

MIT
