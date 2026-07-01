---
name: create-action
description: 为 Uniplat 模型新增 Action（业务操作），修改 JSON 配置 + Groovy 钩子方法
version: 1.0.1
---

# 创建 Uniplat Action

> 本 skill 用于为已有模型添加新的 Action（业务操作，如"审批"、"提交"、"导出"等）。

## 参数解析

`$ARGUMENTS` 应包含：
- **模型文件路径**（必须）：模型 JSON 文件的路径
- **Action 名**（必须）：snake_case，如 `approve`、`submit_review`
- **Action 类型**（可选）：`custom`（默认）、`insert`、`update`、`delete`

示例：
- `path/to/model.json approve` → 添加审批 Action
- `path/to/model.json batch_export custom` → 添加批量导出

## 执行流程

### 1. 定位模型文件

用 Glob 查找用户指定的模型文件：
- `<模型名>.json`
- `<模型名>.groovy`

如果找不到，提示用户先使用 `/uniplat:create-model` 创建模型。

### 2. 确认 Action 信息

用 AskUserQuestion 询问：

```
Action 名：approve
Action 标签：审批
Action 类型：custom
容器类型：dialog（弹窗）/ none（直接执行）
表单参数：
  - status: mapping (status_mapping) 必填
  - comment: text 选填
执行后跳转：refresh / close
```

### 3. 在 JSON 的 action_defs 数组中添加

读取模型 JSON，在 `action_defs` 数组中追加新 Action：

```json
{
  "name": "approve",
  "when": "1",
  "label": "审批",
  "container": "dialog",
  "parameters": {
    "server": [
      {"property": "approved_by", "result": "env.user_id.value"},
      {"property": "approved_time", "result": "env.now.value"}
    ],
    "inputs": [
      {
        "property": "status",
        "label": "审批结果",
        "type": "mapping",
        "mapping": "approve_status_mapping",
        "required": true,
        "span": 24
      },
      {
        "property": "comment",
        "label": "审批意见",
        "type": "text",
        "span": 24
      }
    ]
  },
  "validator": "approve_validator",
  "behavior": "approve_behavior",
  "forward": "refresh",
  "prompt": ""
}
```

**Action 常用属性**：

| 属性 | 类型 | 说明 |
|------|------|------|
| `name` | string | Action 唯一标识（snake_case） |
| `when` | string | 显示条件（`"1"` 总是显示，或表达式如 `"object.status.value as int == 0"`） |
| `label` | string | 按钮文字 |
| `container` | string | `"dialog"` 弹窗表单 / `"none"` 直接执行 / `""` 新页面 |
| `on` | string | `"object"` 表示在数据行上操作（用于列表行操作） |
| `parameters.server` | array | 服务端自动填充参数（表达式） |
| `parameters.inputs` | array | 用户填写的表单参数 |
| `validator` | string | 校验方法名（可选） |
| `behavior` | string | 行为方法名（必须） |
| `forward` | string | 执行后跳转（`"refresh"` / `"close"` / 表达式） |
| `prompt` | string | 执行前确认提示（可选） |
| `tx` | boolean | 是否开启事务（可选） |
| `open_in_new_tab` | boolean | 是否在新标签打开（可选） |
| `size_percent` | number | 弹窗宽度百分比（可选） |

**输入参数常用属性**：

| 属性 | 类型 | 说明 |
|------|------|------|
| `property` | string | 字段名 |
| `label` | string | 标签 |
| `type` | string | `text`/`number`/`mapping`/`date`/`hidden`/`boolean`/`file` |
| `required` | boolean | 是否必填 |
| `default_value` | string | 默认值表达式（如 `object.name.value`） |
| `mapping` | string | type=mapping 时引用的映射名 |
| `updator` | string | 字段联动方法名 |
| `span` | number | 表单列宽（12=半行，24=整行） |
| `is_param` | boolean | 是否作为 URL 参数传递 |

### 4. 在 Groovy 中添加钩子方法

读取模型 Groovy，添加对应的 Validator 和 Behavior：

```groovy
// ====== 审批校验 ======
def approve_validator(ActionValidatorContext ctx) {
    def obj = ctx.dataList.fetchOne()
    def inputs = ctx.inputs

    // 校验逻辑
    AssertUtils.isTrue(inputs.status.value, "请选择审批结果")
    AssertUtils.isFalse(obj.getBoolean("is_del"), "已删除的数据不能审批")

    // 条件校验
    def currentStatus = obj.status.value as int
    AssertUtils.isTrue(currentStatus == 0, "只有待审批状态才能审批")
}

// ====== 审批行为 ======
def approve_behavior(ActionBehaviorContext ctx) {
    def obj = ctx.dataList.fetchOne()
    def inputs = ctx.inputs
    def host = ctx.host
    def now = host.getTimestamp()
    def operator = host.getUser().id as long

    // 业务逻辑
    def status = inputs.status.value as int
    def comment = inputs.comment.value ?: ""

    obj.update([
        status        : status,
        approve_comment: comment,
        approved_by   : operator,
        approved_time : now,
        updated_by    : operator,
        updated_time  : now
    ])

    // 可选：发送事件
    // host.sendModelEvent("model_name", "approved", obj)

    return [
        result: 0,
        id    : obj.getKeyValue(),
        msg   : "审批成功"
    ]
}
```

### 5. 将新 Action 添加到 list 的 row_actions

在 JSON 的 `list.row_actions` 数组中添加新 Action 名：

```json
"list": {
  "row_actions": ["update", "delete", "approve"]
}
```

### 6. 向用户确认

完成后展示：
- Action 配置摘要
- Groovy 钩子方法
- API 端点示例：

```
Action 已添加：
  名称：approve (审批)
  显示条件：when: "1"
  参数：
    - status: mapping (必填)
    - comment: text (选填)
  方法：
    - Validator: approve_validator
    - Behavior: approve_behavior
  已添加到：list.row_actions

前端调用：
  POST /general/model/{model}/action/approve/execute
```

---

## 常见 Action 模板

### 审批类

```json
{
  "name": "approve",
  "when": "object.status.value as int == 0",
  "label": "审批",
  "container": "dialog",
  "parameters": {
    "server": [
      {"property": "approved_by", "result": "env.user_id.value"},
      {"property": "approved_time", "result": "env.now.value"}
    ],
    "inputs": [
      {"property": "approve_status", "label": "结果", "type": "mapping", "mapping": "approve_status_mapping", "required": true},
      {"property": "comment", "label": "意见", "type": "text"}
    ]
  },
  "behavior": "approve_behavior",
  "forward": "refresh"
}
```

### 状态变更类

```json
{
  "name": "publish",
  "when": "object.status.value as int == 0",
  "label": "发布",
  "container": "none",
  "parameters": {
    "server": [
      {"property": "status", "result": "1"},
      {"property": "publish_time", "result": "env.now.value"}
    ],
    "inputs": []
  },
  "behavior": "update",
  "forward": "refresh",
  "prompt": "确认发布？"
}
```

### 软删除类

```json
{
  "name": "delete",
  "when": "1",
  "label": "删除",
  "container": "none",
  "parameters": {
    "server": [
      {"property": "is_del", "result": "1"},
      {"property": "updated_by", "result": "env.user_id.value"},
      {"property": "updated_time", "result": "env.now.value"}
    ],
    "inputs": []
  },
  "behavior": "update",
  "forward": "refresh",
  "prompt": "确认删除？"
}
```

### 设为默认类

```json
{
  "name": "setDefault",
  "when": "!object.is_default.value",
  "on": "object",
  "label": "设为默认",
  "container": "dialog",
  "parameters": {
    "server": [
      {"property": "updated_by", "result": "env.user_id.value"},
      {"property": "updated_time", "result": "env.now.value"}
    ],
    "inputs": [
      {"property": "is_default", "label": "设置默认", "type": "hidden", "default_value": "1"}
    ]
  },
  "behavior": "setDefault",
  "forward": "refresh"
}
```

### 自定义行为类

```json
{
  "name": "sync_data",
  "when": "1",
  "label": "同步数据",
  "container": "none",
  "tx": true,
  "parameters": {
    "server": [],
    "inputs": []
  },
  "behavior": "syncDataBehavior",
  "forward": "refresh",
  "prompt": "确认同步？此操作可能需要较长时间"
}
```

### 跳转类

```json
{
  "name": "view_detail",
  "when": "1",
  "label": "查看详情",
  "container": "",
  "parameters": {
    "server": [],
    "inputs": []
  },
  "forward": "/path/to/detail/{{object.id.value}}",
  "open_in_new_tab": true
}
```

---

## 注意事项

1. **Action name 用 snake_case**：`approve`、`batch_export`
2. **behavior 方法名**：可以是任意名（如 `approve_behavior`、`insertBehavior`）
3. **validator 方法名**：与 behavior 对应（如 `approve_validator`）
4. **when 条件表达式**：用 `object.field.value` 访问当前行数据
5. **soft delete**：删除操作通常通过 `is_del=1` 实现，behavior 指向 `update`
6. **server 参数自动填充**：用户不可见，常用于填充 `updated_by`、`updated_time`
7. **list.row_actions**：Action 必须添加到此处才会在列表行上显示
8. **list.actions**：全局操作按钮（如"新增"）添加到此处
