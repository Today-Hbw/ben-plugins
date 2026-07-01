---
name: uniplat
description: Uniplat 低代码平台开发入口：生成脚手架、查看规范、快速创建模型/Action/服务
argument-hint: <命令> [参数...]
allowed-tools: ["Read", "Write", "Glob", "Grep", "AskUserQuestion", "TodoWrite", "Skill", "Bash"]
---

# Uniplat 低代码平台开发

**你是一个 Uniplat 平台开发助手。** 根据用户的指令，帮助快速开发 Uniplat 低代码平台应用。

## 参数解析

`$ARGUMENTS` 支持以下命令：

### 脚手架生成

| 命令 | 参数 | 说明 |
|------|------|------|
| `model` | `<模型名> [领域路径]` | 创建新模型（JSON + Groovy） |
| `action` | `<模型名> <Action名> [类型]` | 为模型添加 Action |
| `service` | `<domain/model> <服务名> [方法名...]` | 创建领域/模型服务 |

### 信息查询

| 命令 | 参数 | 说明 |
|------|------|------|
| `guide` | 无 | 查看编码规范速查 |
| `structure` | `[领域路径]` | 查看领域目录结构 |
| `models` | `<领域路径>` | 列出领域下所有模型 |

### 示例

```
/uniplat model Order                    # 创建 Order 模型
/uniplat action Order approve           # 给 Order 添加审批 Action
/uniplat service domain User query      # 创建用户领域服务
/uniplat guide                          # 查看平台规范
```

## 执行流程

### 1. 识别命令

根据 `$ARGUMENTS` 的第一个单词判断操作类型：

- `model` → 调用 `uniplat:create-model`
- `action` → 调用 `uniplat:create-action`
- `service` → 调用 `uniplat:create-service`
- `guide` → 加载 `uniplat:coding-guide`，输出规范摘要
- `structure` → 扫描并展示领域目录结构
- `models` → 扫描并列出领域下所有模型

### 2. 无参数时的默认行为

如果 `$ARGUMENTS` 为空，使用 AskUserQuestion 询问用户想做什么：

```
Uniplat 开发助手，我可以帮你：

A. 创建新模型 → 生成 JSON + Groovy 脚手架
B. 添加 Action → 为模型添加业务操作
C. 创建服务 → 创建 DomainService 或 ModelService
D. 查看规范 → 平台编码规范速查
E. 查看项目结构 → 分析当前领域目录
```

### 3. 执行对应 Skill

使用 Skill 工具加载对应的 skill：

```
Skill("uniplat:create-model", args)
Skill("uniplat:create-action", args)
Skill("uniplat:create-service", args)
Skill("uniplat:coding-guide")
```

### 4. structure 命令

扫描并展示领域目录结构：

```groovy
<领域路径>/
├── config.json
├── Entrances.groovy
├── models/
│   ├── Order/
│   │   ├── Order.json
│   │   └── Order.groovy
│   └── Product/
├── services/
│   ├── User.groovy
│   └── BeforeUser.groovy
├── utils/
│   └── SystemOrgUtil.groovy
├── consts/
│   └── base/common.json
└── monitors/
    └── ProcessMonitors.groovy
```

### 5. models 命令

列出领域下所有模型：

```
领域：my_domain
模型列表：
  - Order (t_order)
    - Actions: create, edit, delete, approve
  - Product (t_product)
    - Actions: create, edit, delete
  - User (t_user)
    - Actions: create, edit, delete, reset_password
```

实现方式：
1. Glob 扫描 `models/*/` 目录
2. 读取每个 `ModelName.json` 提取模型信息
3. 解析 actions 字段列出所有 Action

### 6. guide 命令

加载 `uniplat:coding-guide`，输出规范摘要：

**核心要点**：
- 领域目录结构：`models/` `services/` `utils/` `consts/`
- 数据对象：`DataObject`（数据行）/ `PureDataObject`（输入参数）/ `EnvDataObject`（环境）
- Groovy 钩子：`validate` / `doBehavior` / `updator` / `onChange`（全部 static）
- Host 单例：`Host.getInstance()` 获取平台资源
- API 端点：`/general/model/` `/general/project/`

完整规范参见 `uniplat:coding-guide` skill。

---

## 与 dev-flow 协同使用

本插件设计为与 `dev-flow` 研发流程插件**完全解耦**，通过项目配置文件协同：

### 在 Uniplat 项目中使用 dev-flow

1. 在项目根目录创建 `.claude/CLAUDE.md`：

```markdown
# 项目规范

## 研发流程
使用 `/dev-flow` 插件管理完整研发流程。

## 平台规范
本项目基于 Uniplat 低代码平台开发，编码时必须遵守：
- 使用 `/uniplat:create-model` 创建模型
- 使用 `/uniplat:create-action` 创建 Action
- 使用 `/uniplat:create-service` 创建服务
- 遵循 `/uniplat:coding-guide` 中的规范
```

2. 在项目目录下运行 `/dev-flow`，流程会自动识别 Uniplat 项目并按平台规范开发。

### 单独使用 uniplat

不依赖 dev-flow，直接使用本插件的脚手架功能：

```
/uniplat model Order                    # 创建模型
/uniplat action Order approve           # 添加 Action
/uniplat service domain User query      # 创建服务
/uniplat guide                          # 查看规范
```

---

## 输出规范

- 所有生成的文件使用 UTF-8 编码
- JSON 文件使用 2 空格缩进
- Groovy 文件使用 4 空格缩进
- 文件名与类名严格一致（大小写敏感）
- 新增文件后向用户确认，说明文件路径和用途

## 错误处理

- 模型/领域目录不存在 → 提示用户先创建或指定正确路径
- JSON 解析失败 → 展示错误信息，建议用户检查格式
- Groovy 语法错误 → 展示错误位置，提供修复建议
- 参数不足 → 用 AskUserQuestion 询问缺失信息
