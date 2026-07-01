---
name: create-action
description: 为 Uniplat 模型新增 Action（业务操作），修改 JSON 配置 + Groovy 钩子方法
version: 1.0.2
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

## 关键原则

### behavior 与 Groovy 方法的关系

| behavior 值 | 含义 | 是否需要 Groovy 方法 |
|-------------|------|---------------------|
| `"insert"` | 平台内置插入 | **不需要** |
| `"update"` | 平台内置更新 | **不需要** |
| `"delete"` | 平台内置删除 | **不需要** |
| `"myCustomMethod"` | 自定义方法名 | **需要**，在 Groovy 中定义对应方法 |
| `""` | 空，无行为 | 不需要（纯跳转/弹窗类） |

**只有自定义 behavior 才需要在 Groovy 中写方法。** 使用平台内置的 insert/update 不需要。

### when 条件空值安全

当 `when` 使用 `object.field.value` 且字段可能为空时：

```json
// 安全写法（使用 ?. 安全导航）
{"when": "object.status?.value as int == 0"}
{"when": "object.field?.value"}
{"when": "object.status?.value && object.type?.value == 'CORP'"}

// 确定有值的字段可以直接用
{"when": "object.is_del?.value == 0"}
{"when": "!object.is_del?.value"}
```

**注意**：列表行操作（`"on": "object"`）的 when 条件必须考虑空值，因为 object 可能来自不同的数据源。

## 执行流程

### 1. 定位模型文件

用 Glob 查找用户指定的模型文件。如果找不到，提示用户先使用 `/uniplat:create-model` 创建模型。

### 2. 确认 Action 信息

用 AskUserQuestion 询问：

```
Action 名：approve
Action 标签：审批
容器类型：dialog（弹窗）/ none（直接执行）
是否需要自定义 behavior：是/否
  - 否 → 使用平台内置 behavior（insert/update）
  - 是 → 指定自定义方法名
表单参数：...
when 条件：（默认 "1"，或表达式）
```

### 3. 在 JSON 的 action_defs 数组中添加

#### 情况 A：使用平台内置 behavior（简单场景）

```json
{
  "name": "publish",
  "when": "object.status?.value as int == 0",
  "label": "发布",
  "container": "none",
  "parameters": {
    "server": [
      {"property": "status", "result": "1"},
      {"property": "publish_time", "result": "env.now.value"},
      {"property": "updated_by", "result": "env.user_id.value"}
    ],
    "inputs": []
  },
  "behavior": "update",
  "forward": "refresh",
  "prompt": "确认发布？"
}
```

**这种情况下，Groovy 不需要写方法。**

#### 情况 B：使用自定义 behavior（复杂业务逻辑）

```json
{
  "name": "approve",
  "when": "object.status?.value as int == 0",
  "label": "审批",
  "container": "dialog",
  "parameters": {
    "server": [
      {"property": "approved_by", "result": "env.user_id.value"},
      {"property": "approved_time", "result": "env.now.value"}
    ],
    "inputs": [
      {
        "property": "approve_status",
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

**这种情况下，需要在 Groovy 中写 `approve_behavior` 和 `approve_validator` 方法。**

### 4. 在 Groovy 中添加钩子方法（仅自定义 behavior 时）

```groovy
// ====== 审批校验 ======
def approve_validator(ActionValidatorContext ctx) {
    def obj = ctx.dataList.fetchOne()
    def inputs = ctx.inputs

    AssertUtils.isTrue(inputs.approve_status.value, "请选择审批结果")
    AssertUtils.isFalse(obj.getBoolean("is_del"), "已删除的数据不能审批")

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

    def status = inputs.approve_status.value as int
    def comment = inputs.comment.value ?: ""

    obj.update([
        status         : status,
        approve_comment: comment,
        approved_by    : operator,
        approved_time  : now,
        updated_by     : operator,
        updated_time   : now
    ])

    return [result: 0, id: obj.getKeyValue(), msg: "审批成功"]
}
```

### 5. 将新 Action 添加到 list

在 JSON 的 `list.row_actions` 或 `list.actions` 数组中添加新 Action 名：

```json
"list": {
  "row_actions": ["update", "delete", "approve"],
  "actions": ["insert", "publish"]
}
```

### 6. 向用户确认

完成后展示：

```
Action 已添加：
  名称：approve (审批)
  显示条件：object.status?.value as int == 0
  behavior：approve_behavior（自定义方法）

  已在 Groovy 中添加：
    - approve_validator：校验方法
    - approve_behavior：行为方法

  已添加到：list.row_actions
```

---

## 常见 Action 模板

### 状态变更（使用平台内置 update）

```json
{
  "name": "publish",
  "when": "object.status?.value as int == 0",
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

**不需要 Groovy 方法。**

### 软删除（使用平台内置 update）

```json
{
  "name": "delete",
  "when": "!object.is_del?.value",
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

**不需要 Groovy 方法。**

### 审批（自定义 behavior）

```json
{
  "name": "approve",
  "when": "object.status?.value as int == 0",
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
  "validator": "approve_validator",
  "behavior": "approve_behavior",
  "forward": "refresh"
}
```

**需要 Groovy 方法**：`approve_validator` + `approve_behavior`

### 设为默认（自定义 behavior）

```json
{
  "name": "setDefault",
  "when": "!object.is_default?.value",
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

**需要 Groovy 方法**：`setDefault`（需要取消其他默认项的复杂逻辑）

### 跳转类（无 behavior）

```json
{
  "name": "view_detail",
  "when": "1",
  "label": "查看详情",
  "container": "",
  "parameters": {"server": [], "inputs": []},
  "forward": "/path/to/detail/{{object.id.value}}",
  "open_in_new_tab": true
}
```

**不需要 Groovy 方法。**

### 复合条件 when

```json
{
  "name": "submit_audit",
  "when": "object.status?.value as int == 0 && object.audit_status?.value != 2",
  "label": "提交审核",
  "container": "none",
  "parameters": {
    "server": [
      {"property": "audit_status", "result": "1"},
      {"property": "submit_time", "result": "env.now.value"}
    ],
    "inputs": []
  },
  "behavior": "update",
  "forward": "refresh",
  "prompt": "确认提交审核？"
}
```

---

## 注意事项

1. **优先使用平台内置 behavior**：insert/update/delete 不需要 Groovy 方法
2. **只有自定义 behavior 才写 Groovy 方法**
3. **when 条件用 `?.value` 安全导航**：特别是 `on: "object"` 的行操作
4. **复合条件用 `&&` 或 `||`**：注意空值安全
5. **soft delete**：删除操作通常通过 `is_del=1` + `behavior: "update"` 实现
6. **list.row_actions vs list.actions**：行操作放 row_actions，全局操作放 actions
