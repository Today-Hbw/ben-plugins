---
name: create-action
description: 为 Uniplat 模型新增 Action（业务操作），生成 JSON 配置 + Groovy 钩子模板
version: 1.0.0
---

# 创建 Uniplat Action

> 本 skill 用于为已有模型添加新的 Action（业务操作，如"审批"、"提交"、"导出"等）。

## 参数解析

`$ARGUMENTS` 应包含：
- **模型名**（必须）：模型名称，如 `Order`
- **Action 名**（必须）：小写下划线格式，如 `approve`、`submit_review`
- **Action 类型**（可选）：`create` / `update` / `delete` / `custom`，默认 `custom`

示例：
- `Order approve` → 给 Order 模型添加审批 Action
- `Order submit_review custom` → 添加自定义类型的 Action

## 执行流程

### 1. 定位模型文件

根据模型名找到：
- `<领域目录>/models/<ModelName>/<ModelName>.json`
- `<领域目录>/models/<ModelName>/<ModelName>.groovy`

如果找不到，提示用户先使用 `/uniplat:create-model` 创建模型。

### 2. 确认 Action 信息

用 AskUserQuestion 询问（如果参数未提供）：

```
模型：Order
Action 名：approve
Action 标签：审批
Action 类型：custom
表单参数：[status（下拉框）、comment（文本域）]
```

### 3. 在 JSON 中添加 Action 配置

读取 `<ModelName>.json`，在 `actions` 对象中添加新 Action：

```json
{
  "actions": {
    "approve": {
      "label": "审批",
      "type": "custom",
      "icon": "check",
      "parameters": [
        {
          "name": "status",
          "label": "审批结果",
          "type": "select",
          "required": true,
          "options": [
            { "value": "approved", "label": "通过" },
            { "value": "rejected", "label": "驳回" }
          ],
          "updator": "updator"
        },
        {
          "name": "comment",
          "label": "审批意见",
          "type": "textarea",
          "required": false
        }
      ],
      "variables": [
        {
          "name": "need_reject_reason",
          "defaultValue": false
        }
      ]
    }
  }
}
```

**Action 配置字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `label` | string | 按钮显示文字 |
| `type` | string | `create` / `update` / `delete` / `custom` |
| `icon` | string | 图标名（可选） |
| `parameters` | array | 表单参数列表 |
| `variables` | array | Action 状态变量（可选） |
| `enabled` | string | 启用条件表达式（可选） |
| `whenPassed` | string | 条件检查表达式（可选） |
| `authPassed` | string | 权限检查表达式（可选） |

**参数类型**：

| type | 前端组件 | 额外字段 |
|------|---------|---------|
| `input` | 单行文本 | `placeholder`, `maxLength` |
| `textarea` | 多行文本 | `placeholder`, `rows` |
| `select` | 下拉框 | `options`（数组） |
| `radio` | 单选组 | `options` |
| `checkbox` | 复选框 | — |
| `number` | 数字输入 | `min`, `max`, `step` |
| `date` | 日期选择 | `format` |
| `datetime` | 日期时间选择 | `format` |
| `reference` | 关联选择 | `model`, `labelField` |
| `upload` | 文件上传 | `accept`, `multiple` |

**参数联动（updator）**：

如果参数有联动逻辑，指定 `updator` 字段名（对应 Groovy 中的方法名）：

```json
{
  "name": "status",
  "updator": "status_updator"
}
```

Groovy 中对应方法：

```groovy
static void status_updator(ParameterUpdateMasterContext ctx) {
    String sender = ctx.getSender()
    if (sender == "status") {
        String status = ctx.getParameters().find { it.name == "status" }.value
        if (status == "rejected") {
            ctx.setData("need_reject_reason", true)
        }
    }
}
```

### 4. 在 Groovy 中添加钩子方法

读取 `<ModelName>.groovy`，添加对应的 Validator 和 Behavior：

```groovy
/**
 * 审批校验
 */
static String validate_approve(ActionValidatorContext ctx) {
    DataObject row = ctx.row
    PureDataObject inputs = ctx.inputs

    String status = inputs.getString("status")
    if (!status) {
        return "请选择审批结果"
    }

    if (status == "rejected" && !inputs.getString("comment")) {
        return "驳回时必须填写审批意见"
    }

    return null
}

/**
 * 审批业务逻辑
 */
static Object doBehavior_approve(ActionBehaviorContext ctx) {
    DataObject row = ctx.row
    PureDataObject inputs = ctx.inputs
    EnvDataObject env = ctx.env

    String status = inputs.getString("status")
    String comment = inputs.getString("comment")

    // 更新状态
    row.put("status", status)
    row.put("approve_comment", comment)
    row.put("approved_by", env.getUserId())
    row.put("approved_at", new Date())
    row.update()

    // 可选：发送事件通知
    // Host.getInstance().sendModelEvent("Order", "approved", row)

    return [success: true, message: "审批完成"]
}
```

**钩子方法命名规则**：

| 方法名格式 | 用途 |
|-----------|------|
| `validate` | 全局 Validator（所有 Action 共用） |
| `validate_<actionName>` | 特定 Action 的 Validator |
| `doBehavior` | 全局 Behavior（所有 Action 共用） |
| `doBehavior_<actionName>` | 特定 Action 的 Behavior |
| `updator` | 全局 Updator |
| `updator_<actionName>` | 特定 Action 的 Updator |
| `onChange` | 全局 onChange |
| `onChange_<actionName>` | 特定 Action 的 onChange |

### 5. 向用户确认

生成完成后，展示：
- Action 配置摘要（名称、类型、参数列表）
- Groovy 钩子方法列表
- API 端点示例：

```
前端调用示例：
POST /general/model/Order/action/approve/execute
Body: { "id": "123", "status": "approved", "comment": "同意" }

property2（获取表单）：
POST /general/model/Order/action/approve/property2
Body: { "id": "123" }
```

---

## 常见 Action 模板

### 审批类

```json
{
  "label": "审批",
  "type": "custom",
  "parameters": [
    { "name": "status", "label": "结果", "type": "select", "required": true,
      "options": [{"value": "approved", "label": "通过"}, {"value": "rejected", "label": "驳回"}] },
    { "name": "comment", "label": "意见", "type": "textarea" }
  ]
}
```

### 状态变更类

```json
{
  "label": "发布",
  "type": "custom",
  "parameters": [
    { "name": "publish_at", "label": "发布时间", "type": "datetime" }
  ]
}
```

### 批量操作类

```json
{
  "label": "批量导出",
  "type": "custom",
  "batch": true,
  "parameters": [
    { "name": "format", "label": "格式", "type": "select",
      "options": [{"value": "xlsx", "label": "Excel"}, {"value": "csv", "label": "CSV"}] }
  ]
}
```

### 工作流启动类

```json
{
  "label": "发起审批",
  "type": "custom",
  "startProcess": "order_approve_flow",
  "parameters": [
    { "name": "reason", "label": "申请原因", "type": "textarea", "required": true }
  ]
}
```

---

## 注意事项

1. **Action 名必须小写下划线**：`approve`、`submit_review`，不能是 `Approve` 或 `approveReview`
2. **特定 Action 的钩子方法名带后缀**：`validate_approve`，不是 `validateApprove`
3. **parameters 是数组**：JSON 中 `parameters` 必须是数组，不是对象
4. **options 格式**：`[{"value": "xxx", "label": "显示文字"}]`
5. **variables 可选**：用于跨字段联动的中间状态，存在 Redis 中
6. **batch: true**：批量操作时需声明，平台会传入多条数据
