---
name: create-model
description: 在指定子项目目录下创建 Uniplat 数据模型脚手架（JSON 元数据 + Groovy 钩子文件）
version: 1.0.3
---

# 创建 Uniplat 数据模型

> 本 skill 用于在 Uniplat 子项目中快速创建一个新的数据模型，生成 `<model_name>.json` + `<model_name>.groovy` 脚手架。

## 参数解析

`$ARGUMENTS` 应包含：
- **模型名**（必须）：表名格式（snake_case），如 `system_user`、`pay_order`；或 CamelCase 如 `UniplatChat`
- **子项目路径**（可选）：子项目目录路径，省略则询问用户
- **分组目录**（可选）：`models/` 下的分组子目录，如 `system`、`business/order`
- **数据库源**（可选）：默认 `(host)`，可指定 `mssql_finance` 等

示例：
- `pay_order` → 创建 pay_order 模型
- `HRO_BalanceRefundOrder /path/to/subproject business/order` → 在指定分组创建

## 执行流程

### 1. 解析参数并确认

如果参数不完整，用 AskUserQuestion 询问：

```
模型名/表名：______
子项目目录：______（从项目结构推断，或让用户选择）
分组目录（models/下）：______（如 system、business/order）
数据库源：______（默认 (host)）
模型描述：______（中文描述）
是否需要自定义 insert/update 逻辑：是/否（默认否，使用平台内置）
```

### 2. 创建目录结构

```
<子项目>/models/<分组>/<model_name>.json
<子项目>/models/<分组>/<model_name>.groovy
```

**注意**：不创建 test 目录（test 是废弃机制）。

### 3. 生成 JSON 元数据文件

按以下模板生成。**关键点**：`behavior: "insert"`、`behavior: "update"` 使用平台内置实现，**不需要在 Groovy 中写对应方法**。

```json
{
  "database": "(host)",
  "table": "{{table_name}}",
  "modelDescription": "{{模型描述}}",
  "key_field": "id",
  "data_right": false,

  "mapping_defs": [
    {
      "name": "status_mapping",
      "mapping_values": [
        {"key": "0", "value": "草稿"},
        {"key": "1", "value": "正常"},
        {"key": "-1", "value": "已删除"}
      ]
    }
  ],
  "mapping_refs": [],
  "joint_defs": [],
  "field_defs": [
    {
      "property": "id",
      "label": "id",
      "type": "number",
      "format": "0",
      "column": {
        "type": "BIGINT",
        "autoIncrement": true,
        "comment": "主键"
      }
    },
    {
      "property": "name",
      "label": "名称",
      "type": "text",
      "column": {
        "type": "VARCHAR",
        "length": 100,
        "nullable": false,
        "comment": "名称"
      }
    },
    {
      "property": "status",
      "label": "状态",
      "type": "mapping",
      "mapping": "status_mapping",
      "column": {
        "type": "INT",
        "nullable": false,
        "defaultValue": "0",
        "comment": "状态"
      }
    },
    {
      "property": "created_time",
      "label": "创建时间",
      "type": "date",
      "format": "yyyy-MM-dd HH:mm:ss",
      "column": {
        "type": "DATETIME",
        "nullable": true,
        "defaultValue": "current_timestamp()",
        "comment": "创建时间"
      }
    },
    {
      "property": "updated_time",
      "label": "更新时间",
      "type": "date",
      "format": "yyyy-MM-dd HH:mm:ss",
      "column": {
        "type": "DATETIME",
        "nullable": true,
        "defaultValue": "current_timestamp()",
        "comment": "更新时间"
      }
    },
    {
      "property": "created_by",
      "label": "创建人",
      "type": "number",
      "format": "0",
      "column": {
        "type": "BIGINT",
        "nullable": true,
        "defaultValue": "0",
        "comment": "创建人ID"
      }
    },
    {
      "property": "updated_by",
      "label": "更新人",
      "type": "number",
      "format": "0",
      "column": {
        "type": "BIGINT",
        "nullable": true,
        "defaultValue": "0",
        "comment": "更新人ID"
      }
    },
    {
      "property": "is_del",
      "label": "删除标记",
      "type": "number",
      "format": "0",
      "column": {
        "type": "INT",
        "nullable": false,
        "defaultValue": "0",
        "comment": "0:正常 1:已删除"
      }
    }
  ],
  "calculator_defs": [],
  "action_defs": [
    {
      "name": "insert",
      "when": "1",
      "label": "新增",
      "container": "dialog",
      "parameters": {
        "server": [
          {"property": "created_by", "result": "env.user_id.value"},
          {"property": "created_time", "result": "env.now.value"},
          {"property": "updated_by", "result": "env.user_id.value"},
          {"property": "updated_time", "result": "env.now.value"}
        ],
        "inputs": [
          {
            "property": "name",
            "label": "名称",
            "type": "text",
            "required": true,
            "span": 12
          },
          {
            "property": "status",
            "label": "状态",
            "type": "mapping",
            "mapping": "status_mapping"
          }
        ]
      },
      "behavior": "insert",
      "forward": "refresh",
      "prompt": ""
    },
    {
      "name": "update",
      "when": "1",
      "label": "修改",
      "container": "dialog",
      "parameters": {
        "server": [
          {"property": "updated_by", "result": "env.user_id.value"},
          {"property": "updated_time", "result": "env.now.value"}
        ],
        "inputs": [
          {
            "property": "name",
            "label": "名称",
            "type": "text",
            "default_value": "object.name.value",
            "required": true,
            "span": 12
          },
          {
            "property": "status",
            "label": "状态",
            "type": "mapping",
            "mapping": "status_mapping",
            "default_value": "object.status.value"
          }
        ]
      },
      "behavior": "update",
      "forward": "refresh",
      "prompt": ""
    },
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
  ],
  "references": [],
  "list": {
    "label": "{{模型描述}}列表",
    "filters": [
      {"label": "名称", "field": "name", "type": "text"},
      {"label": "状态", "field": "status", "type": "enum", "mapping": "status_mapping"}
    ],
    "actions": ["insert"],
    "row_actions": ["update", "delete"],
    "field_groups": [
      {"label": "名称", "template": "{name}"},
      {"label": "状态", "template": "{status}"},
      {"label": "创建时间", "template": "{created_time}"}
    ],
    "detail_action_visible": false,
    "dataFilters": [
      {"field": "is_del", "value": "0"}
    ]
  },
  "detail": {
    "label": "",
    "title_template": "",
    "actions": [],
    "header": {"field_groups": [], "actions": []},
    "pages": []
  },
  "modelVersion": false
}
```

### 4. 生成 Groovy 文件

**关键原则**：如果 insert/update/delete 使用平台内置 behavior（`behavior: "insert"`、`behavior: "update"`），**不需要在 Groovy 中写对应方法**。只有自定义 behavior 才需要写方法。

**情况 A：使用平台内置 behavior（默认）**

```groovy
package models.{{group}}

import com.qinqinxiaobao.report.uniplat.engine.DO.DataObject
import com.qinqinxiaobao.report.uniplat.executor.ParameterUpdateMasterContext
import com.qinqinxiaobao.report.uniplat.executor.ParameterChangeContext
import com.qinqinxiaobao.report.uniplat.host.Host

class {{model_name}} {
    def model_name = "{{model_name}}"

    // ====== Updator（字段联动） ======
    // 仅当 JSON 中 input 配置了 "updator": "xxx_updator" 时才需要

    def status_updator(ParameterUpdateMasterContext context) {
        if (!context.sender || context.sender == "type") {
            return [default_value: "1"]
        }
    }

    // ====== onChange（字段变更回调） ======

    def onStatusChange(ParameterChangeContext context) {
        def status = context.params.status
        return [variables: [show_detail: status == "1"]]
    }

    // ====== 自定义业务方法 ======
    // 仅当 action_defs 中 behavior 指向自定义方法名时才需要
}
```

**情况 B：需要自定义 insert/update 逻辑**

如果用户在第 1 步选择"需要自定义 insert/update 逻辑"，则生成完整模板：

```groovy
package models.{{group}}

import com.qinqinxiaobao.report.uniplat.engine.DO.DataObject
import com.qinqinxiaobao.report.uniplat.executor.ActionBehaviorContext
import com.qinqinxiaobao.report.uniplat.executor.ActionValidatorContext
import com.qinqinxiaobao.report.uniplat.executor.BehaviorResult
import com.qinqinxiaobao.report.uniplat.executor.ParameterUpdateMasterContext
import com.qinqinxiaobao.report.uniplat.executor.ParameterChangeContext
import com.qinqinxiaobao.report.uniplat.host.Host
import com.qinqinxiaobao.report.utils.AssertUtils

class {{model_name}} {
    def model_name = "{{model_name}}"

    // ====== Validator（可选） ======
    // 仅当 action_defs 中配置了 "validator": "xxx_validator" 时才需要

    def insert_validator(ActionValidatorContext ctx) {
        def inputs = ctx.inputs
        AssertUtils.isTrue(inputs.name.value, "名称不能为空")
    }

    // ====== Behavior（仅当 behavior 不是 "insert"/"update" 内置名时） ======

    def customInsertBehavior(ActionBehaviorContext ctx) {
        def inputs = ctx.inputs
        def host = ctx.host
        def now = host.getTimestamp()
        def operator = host.getUser().id as long

        // TODO: 添加自定义业务逻辑

        def id = ctx.dataModel.insertByMap([
            name         : inputs.name.value,
            status       : inputs.status.value as int,
            created_by   : operator,
            created_time : now,
            updated_by   : operator,
            updated_time : now,
            is_del       : 0
        ]).id

        return [result: 0, id: id, msg: "添加成功"]   // 或 return new BehaviorResult(0, "添加成功", id)
    }
}
```

### 5. 向用户确认

生成文件后，向用户展示：

```
模型已创建：
  JSON：models/{{group}}/{{model_name}}.json
  Groovy：models/{{group}}/{{model_name}}.groovy

  Action 配置：
    - insert (新增) → behavior: "insert" (平台内置)
    - update (修改) → behavior: "update" (平台内置)
    - delete (删除) → behavior: "update" (软删除，平台内置)

  列表默认过滤：is_del = 0

  Groovy 中未生成 insert/update 方法（使用平台内置）
  如需自定义逻辑，请修改 JSON 中的 behavior 为自定义方法名
```

---

## 重要约定

### behavior 字段说明

| behavior 值 | 含义 | 是否需要 Groovy 方法 |
|-------------|------|---------------------|
| `"insert"` | 平台内置插入 | **不需要** |
| `"update"` | 平台内置更新 | **不需要** |
| `"delete"` | 平台内置删除 | **不需要** |
| `"customMethod"` | 自定义方法 | **需要**，Groovy 中定义 `def customMethod(ActionBehaviorContext ctx)` |
| `""` | 空字符串，无行为 | 不需要（用于纯跳转、弹窗类） |

### when / enable 条件中的空值安全

`when`=是否**显示**，`enable`（=`enabled`）=是否**启用**。二者引用 `object` 时**先 `object != null &&`**（列表级/无选中行时 object 为 null，直接点属性会 NPE）：

```json
// ✅ 先守卫 object 非空
{"when":   "object != null && object.getValue('status') == 0"}
{"enable": "object != null && object.getValue('status') == 0"}
// 关联/joint 字段：object.get('本地键#joint名.字段').value
// 常量条件无需守卫
{"when": "1"}

// ❌ object 可能为空时直接点属性 → NPE
{"when": "object.status.value == 0"}
```

取值：`object.getValue('字段')`（推荐）或 `object.字段.value`（可空字段加 `?.`）。

### dataFilters 说明

`dataFilters` 用于自动过滤列表/详情数据，常用于：
- 软删除过滤：`{"field": "is_del", "value": "0"}`
- 组织隔离：`{"field": "xid", "value": "context.xid"}`
- 状态过滤：`{"field": "valid", "value": "1"}`

可出现在以下位置：
- 模型顶层 `dataFilters`：对所有查询生效
- `list.dataFilters`：仅对列表查询生效
- `list.pages[].dataFilters`：对特定页签生效

### test 目录已废弃

**不要创建 test 目录**。这是历史遗留机制，新项目不使用（增删改查也无需 test）。

### 字段类型（type）全枚举

`field_defs[].type` 的取值（据 `datamodel.json` 的 `T_DATA_TYPE`）：

```
boolean, number, double, money, mapping, cascader, tree,
date, datetime, text, longText, file, image, multi_file,
multi_image, dynamic, enum, grid, search, multi_search,
checkbox-group, hidden
```

- 常用：`text` / `number` / `double` / `money` / `mapping` / `date` / `datetime` / `longText` / `file` / `image` / `cascader` / `tree`。
- ⚠️ Schema 自注：`enum`、`search`、`checkbox-group`、`hidden` 已标记废弃；`dynamic` 待新支持——新模型别用这几个。
- `column` 定义可选：虚拟模型或不落库字段可不写 `column`。

### action 高级字段：on / tx / enable / variables

除 `name/when/label/container/parameters/behavior/forward/prompt` 外，action 常用：

| 字段 | 取值 | 说明 |
|------|------|------|
| `on` | `object`/`list`/`none`/`each` | 作用域：单条行 / 整表 / 全局 / 逐条 |
| `tx` | `true`/`false` | 是否容器自动管理事务 |
| `enable`(=`enabled`) | 表达式 | 是否启用（与 `when` 显示区分）；引用 object 先 `object != null &&` |
| `variables` | `[{name,defaultValue}]` | action 级状态变量，property2 算初值、execute 时 `ctx.variables` 可读 |
| `validator` | 方法名 | 指定校验方法 |
| `customInitFunc` | 方法名 | 动态生成/改写 ActionDef |
| `open_in_new_tab` | `true`/`false` | forward 后新标签打开 |

### 顶层字段速查（除已示范的常规字段外）

| 字段 | 说明 |
|------|------|
| `sql` | 虚拟模型：不绑实体表，用 SQL 定义（仅查询） |
| `eventSubs` | 模型事件订阅：`[{action, handler}]` |
| `group_sums` | 列表合计行的聚合字段 |
| `objectTitle` | 对象标题模板 |
| `mini_detail` | 简略详情（悬停展示） |
| `mapping_refs` / `joint_refs` | 引用外部 mapping / joint 定义 |

### mapping_defs / joint_defs / detail 的增强写法

```json
// mapping_defs：除 mapping_values，还可用 sql 或自定义函数
{"name": "city_mapping", "database": "(host)", "sql": "SELECT id, name FROM district WHERE is_del=0"}
{"name": "app_mapping", "custom_func": "app_mapping_func"}

// joint_defs：除 sql，还可用自定义函数 + 搜索配置
{
  "name": "user", "custom_function": "user_joint",
  "custom_function_search": [{"key": "mobile", "label": "手机"}, {"key": "name", "label": "姓名"}],
  "key_field": "puid", "joint_field": "puid", "field_defs": [ ... ]
}

// detail：pages 可嵌套 list（从表），detail 根级可有 sections
"detail": {
  "pages": [{"name": "orders", "label": "订单", "sections": [], "list": { ... }}],
  "sections": [ ... ]
}
```

---

## 注意事项

1. **默认使用平台内置 behavior**：insert/update/delete 不需要写 Groovy 方法
2. **dataFilters 用于自动过滤**：特别是 `is_del = 0` 软删除过滤
3. **`when`/`enable` 引用 object 时先 `object != null &&`**：取值优先 `object.getValue('字段')`
4. **不创建 test 目录**：这是废弃机制
5. **参考相似模型**：生成前先用 Glob 找类似的现有模型参考
6. **自定义 behavior 返回 `BehaviorResult`（推荐）或字典**，需 import BehaviorResult
