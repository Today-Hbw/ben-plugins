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
| `model` | `<模型名> [子项目路径] [分组]` | 创建新模型（JSON + Groovy） |
| `action` | `<模型文件路径> <Action名>` | 为模型添加 Action |
| `service` | `<服务名> [子项目路径]` | 创建领域服务 |

### 信息查询

| 命令 | 参数 | 说明 |
|------|------|------|
| `guide` | 无 | 查看编码规范速查 |
| `structure` | `[子项目路径]` | 查看子项目目录结构 |
| `models` | `<子项目路径>` | 列出子项目下所有模型 |
| `schema` | 无 | 打开 JSON Schema 参考 |

### 示例

```
/uniplat model system_user                    # 创建 system_user 模型
/uniplat action path/to/model.json approve    # 给模型添加审批 Action
/uniplat service passport_api                 # 创建领域服务
/uniplat guide                                # 查看平台规范
/uniplat schema                               # 查看 JSON Schema 路径
```

## 执行流程

### 1. 识别命令

根据 `$ARGUMENTS` 的第一个单词判断操作类型：

- `model` → 调用 `uniplat:create-model`
- `action` → 调用 `uniplat:create-action`
- `service` → 调用 `uniplat:create-service`
- `guide` → 加载 `uniplat:coding-guide`，输出规范摘要
- `structure` → 扫描并展示子项目目录结构
- `models` → 扫描并列出子项目下所有模型
- `schema` → 输出 JSON Schema 文件路径

### 2. 无参数时的默认行为

如果 `$ARGUMENTS` 为空，使用 AskUserQuestion 询问用户想做什么：

```
Uniplat 开发助手，我可以帮你：

A. 创建新模型 → 生成 JSON + Groovy 脚手架
B. 添加 Action → 为模型添加业务操作
C. 创建服务 → 创建 DomainService
D. 查看规范 → 平台编码规范速查
E. 查看项目结构 → 分析当前子项目目录
F. 查看 JSON Schema → 模型 JSON Schema 路径
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

扫描并展示子项目目录结构：

```
<子项目>/
├── Entrances.groovy
├── consts/
│   └── common.json
├── models/
│   ├── system/              ← 模型分组
│   │   ├── system_user.json
│   │   └── system_user.groovy
│   └── business/
│       └── order/
├── services/                ← 领域服务
│   └── passport_api.groovy
├── monitors/
└── dashboard/
```
（不含 test 目录：test 是废弃机制，不创建）

### 5. models 命令

列出子项目下所有模型：

```
子项目：hro_spview
模型列表：
  - system_user (system_user)
    - Actions: insert, update, delete
    - 分组：system
  - HRO_EnterpriseBalanceRefundOrder (HRO_BalanceRefundOrder)
    - Actions: rst_account_detail_list, rst_account_reception
    - 分组：business/order
  ...
```

实现方式：
1. Glob 扫描 `models/**/*.json`
2. 读取每个 JSON 提取 `table`、`action_defs`、`modelDescription`
3. 按分组目录组织输出

### 6. guide 命令

加载 `uniplat:coding-guide`，输出规范摘要：

**核心要点**：
- 项目结构：子项目（独立 Git） + models/ + services/ + consts/（**不创建 test 目录**，增删改查也无需 test）
- JSON 格式：`field_defs`（type 有全枚举）、`action_defs`、`list`/`detail`、`mapping_defs`、`dataFilters`；action 还有 `on`(作用域)/`tx`(事务)/`enable`/`variables`；顶层还有 `sql`虚拟模型/`eventSubs` 等
- Groovy 格式：实例方法（非 static）、`def` 声明、`package` **可选**（建议按分组目录）
- **默认用平台内置 behavior**：insert/update/delete 不需要 Groovy 方法，只有自定义 behavior 才写
- 钩子方法：Validator（AssertUtils/ValidateException 抛异常）、Behavior（返回 **BehaviorResult** 或字典）、Updator（返回 Map）；另有 page_values_func、mapping/joint 自定义函数等
- **`when`（显示）vs `enable`（启用）**：引用 object 时先 **`object != null &&`**
- Context 类：Action(Validator/Behavior/CustomInit)Context、ParameterUpdate(Master/Detail)Context、ParameterChangeContext、ModelEventContext、ActionPageValuesFuncContext、FilterUpdateContext、GatewayContext/ModelServiceContext 等
- 数据访问三类：DataModel（`queryDataList/getByKeyField/insertByMap/insert/update/deleteByMap/addLog/addRemark`）、DataObject（`getValue/getInt/getLong/get/update/delete/referTo`）、DataList（`fetchOne/one/asList/count/asStream`）
- 获取过滤值：`masterParams`（ActionPageValuesFuncContext）/`getFilterValues`（FilterUpdateContext），**不在** ActionBehaviorContext
- 服务端点：领域服务 7 种（标准/匿名/内部/SOA/流式/匿名流式/MVC）+ 模型服务；匿名在 service 后加 `anonymous`
- 事件处理：`ModelEventContext` + `eventSubs` JSON 配置，取 host 用 `ctx.getHost()` 或 `Host.getInstance()`

完整规范参见 `uniplat:coding-guide` skill。

### 7. schema 命令

输出 JSON Schema 文件路径：

```
JSON Schema 文件位置：
  模型 JSON Schema：uniplat-main/config/schemas/datamodel.json
  应用 JSON Schema：uniplat-main/config/schemas/application.schema.json
  入口 Schema：uniplat-main/config/schemas/entrances.schema.json
  看板 Schema：uniplat-main/config/schemas/dashboard.schema.json

使用 IDE 的 JSON Schema 功能可获得语法提示。
```

---

## 与 dev-flow 协同使用

本插件设计为与 `dev-flow` 研发流程插件**完全解耦**，通过项目配置文件协同：

### 在 Uniplat 项目中使用 dev-flow / dev-flow-lite

1. 在工作区或子项目根目录创建 `.claude/CLAUDE.md`：

```markdown
# 项目规范

## 研发流程
使用 `/dev-flow` 插件管理完整研发流程。
小需求也可使用 `/dev-flow-lite`（4 步精简流程，不写 MD 文件）。

## 平台规范
本项目基于 Uniplat 低代码平台开发，编码时必须遵守：
- 使用 `/uniplat:create-model` 创建模型
- 使用 `/uniplat:create-action` 创建 Action
- 使用 `/uniplat:create-service` 创建服务
- 遵循 `/uniplat:coding-guide` 中的规范
- JSON Schema 参考：uniplat-main/config/schemas/datamodel.json
```

2. 在项目目录下运行 `/dev-flow` 或 `/dev-flow-lite`，流程会自动识别 Uniplat 项目并按平台规范开发。

### 单独使用 uniplat

不依赖 dev-flow，直接使用本插件的脚手架功能：

```
/uniplat model system_user                    # 创建模型
/uniplat action path/to/model.json approve    # 添加 Action
/uniplat service passport_api                 # 创建服务
/uniplat guide                                # 查看规范
```

---

## 输出规范

- JSON 文件使用 2 空格缩进
- Groovy 文件使用 4 空格缩进
- Groovy 文件必须有 `package` 声明
- 文件名与类名严格一致（大小写敏感）
- 新增文件后向用户确认，说明文件路径和用途

## 错误处理

- 模型/子项目目录不存在 → 提示用户先创建或指定正确路径
- JSON 解析失败 → 展示错误信息，建议用户检查格式
- Groovy 语法错误 → 展示错误位置，提供修复建议
- 参数不足 → 用 AskUserQuestion 询问缺失信息
