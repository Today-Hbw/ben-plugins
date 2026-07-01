---
name: create-model
description: 在指定子项目目录下创建 Uniplat 数据模型脚手架（JSON 元数据 + Groovy 钩子文件）
version: 1.0.1
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
```

### 2. 创建目录结构

```
<子项目>/models/<分组>/<model_name>.json
<子项目>/models/<分组>/<model_name>.groovy
```

### 3. 生成 JSON 元数据文件

按以下模板生成。根据用户描述的字段需求填充 `field_defs`、`action_defs`、`list`。

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
      "behavior": "insertBehavior",
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
      "behavior": "updateBehavior",
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
    "detail_action_visible": false
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

按以下模板生成：

```groovy
package models.{{group}}

import com.qinqinxiaobao.report.uniplat.engine.DO.DataObject
import com.qinqinxiaobao.report.uniplat.engine.DO.EnvDataObject
import com.qinqinxiaobao.report.uniplat.executor.ActionBehaviorContext
import com.qinqinxiaobao.report.uniplat.executor.ActionValidatorContext
import com.qinqinxiaobao.report.uniplat.executor.ParameterUpdateMasterContext
import com.qinqinxiaobao.report.uniplat.executor.ParameterChangeContext
import com.qinqinxiaobao.report.uniplat.host.Host
import com.qinqinxiaobao.report.utils.AssertUtils

class {{model_name}} {
    def model_name = "{{model_name}}"

    // ====== Validator ======

    def insertValidator(ActionValidatorContext ctx) {
        def inputs = ctx.inputs
        // TODO: 添加校验逻辑
        // AssertUtils.isTrue(inputs.name.value, "名称不能为空")
    }

    def updateValidator(ActionValidatorContext ctx) {
        def inputs = ctx.inputs
        // TODO: 添加校验逻辑
    }

    // ====== Behavior ======

    def insertBehavior(ActionBehaviorContext ctx) {
        def inputs = ctx.inputs
        def host = ctx.host
        def now = host.getTimestamp()
        def operator = host.getUser().id as long

        // TODO: 添加业务逻辑

        def id = ctx.dataModel.insertByMap([
            name         : inputs.name.value,
            status       : inputs.status.value as int,
            created_by   : operator,
            created_time : now,
            updated_by   : operator,
            updated_time : now,
            is_del       : 0
        ]).id

        return [
            result: 0,
            id    : id,
            msg   : "添加成功"
        ]
    }

    def updateBehavior(ActionBehaviorContext ctx) {
        def inputs = ctx.inputs
        def obj = ctx.dataList.fetchOne()
        def host = ctx.host
        def now = host.getTimestamp()
        def operator = host.getUser().id as long

        // TODO: 添加业务逻辑

        obj.update([
            name         : inputs.name.value,
            status       : inputs.status.value as int,
            updated_by   : operator,
            updated_time : now
        ])

        return [
            result: 0,
            id    : obj.getKeyValue(),
            msg   : "修改成功"
        ]
    }

    // ====== Updator（字段联动） ======

    def status_updator(ParameterUpdateMasterContext context) {
        // TODO: 添加联动逻辑
        // 示例：
        // if (!context.sender || context.sender == "type") {
        //     return [default_value: "1"]
        // }
    }

    // ====== onChange（字段变更回调） ======

    def onStatusChange(ParameterChangeContext context) {
        // TODO: 添加 onChange 逻辑
        // 示例：
        // def status = context.params.status as String
        // return [variables: [show_detail: status == "1"]]
    }
}
```

### 5. 向用户确认

生成文件后，向用户展示：
- 创建的文件路径
- JSON 中定义的字段和 Action
- Groovy 中的钩子方法说明

用 AskUserQuestion 询问是否需要调整字段、添加更多 Action 等。

---

## 注意事项

1. **文件名与表名一致**：`system_user.json` / `system_user.groovy`
2. **package 声明对应分组目录**：`models/system/system_user.groovy` → `package models.system`
3. **Groovy 类名与文件名一致**：`class system_user`
4. **所有钩子方法都是实例方法**（不是 static），用 `def` 声明
5. **JSON 使用 `field_defs` 数组**（不是 `columns` 对象）
6. **Action 参数分 server 和 inputs**：server 自动填充，inputs 用户填写
7. **删除操作通常用软删除**：设置 `is_del=1`，behavior 用 `update`
8. **参考相似模型**：生成前先用 Glob 找类似的现有模型参考
