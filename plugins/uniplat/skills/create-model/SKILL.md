---
name: create-model
description: 在指定领域目录下创建 Uniplat 数据模型脚手架（JSON 元数据 + Groovy 钩子文件）
version: 1.0.0
---

# 创建 Uniplat 数据模型

> 本 skill 用于在 Uniplat 项目中快速创建一个新的数据模型，生成 `ModelName.json` + `ModelName.groovy` 脚手架。

## 参数解析

`$ARGUMENTS` 应包含：
- **模型名**（必须）：大驼峰格式，如 `Order`、`PartyMember`
- **领域路径**（可选）：领域目录路径，省略则询问用户
- **表名**（可选）：数据库表名，省略则自动推断（`t_` + 下划线转换）
- **数据源**（可选）：默认 `default`

示例：
- `Order` → 创建 Order 模型
- `PartyMember /path/to/domain` → 在指定领域创建
- `Order t_order mysql` → 指定表名和数据源

## 执行流程

### 1. 解析参数并确认

如果参数不完整，用 AskUserQuestion 询问：

```
模型名：______
领域目录：______（从项目结构推断，或让用户选择）
数据库表名：t_______（自动从模型名转换）
数据源：______（默认 default）
```

### 2. 创建目录结构

```
<领域目录>/models/<ModelName>/
├── <ModelName>.json
└── <ModelName>.groovy
```

### 3. 生成 JSON 元数据文件

按以下模板生成 `<ModelName>.json`。根据用户描述的字段需求填充 `columns`、`actions`、`views`。

```json
{
  "name": "{{ModelName}}",
  "table": "{{table_name}}",
  "datasource": "{{datasource}}",
  "label": "{{中文显示名}}",
  "key": "id",
  "columns": {
    "id": {
      "label": "ID",
      "type": "number",
      "primary": true,
      "autoIncrement": true
    },
    "created_at": {
      "label": "创建时间",
      "type": "datetime",
      "default_value": "now()"
    },
    "updated_at": {
      "label": "更新时间",
      "type": "datetime",
      "default_value": "now()"
    },
    "created_by": {
      "label": "创建人",
      "type": "string"
    }
  },
  "actions": {
    "create": {
      "label": "新建",
      "type": "create",
      "parameters": [
        {
          "name": "name",
          "label": "名称",
          "type": "input",
          "required": true
        }
      ]
    },
    "edit": {
      "label": "编辑",
      "type": "update",
      "parameters": []
    },
    "delete": {
      "label": "删除",
      "type": "delete"
    }
  },
  "views": {
    "list": {
      "label": "列表",
      "type": "list",
      "columns": ["id", "name", "created_at"],
      "actions": ["create", "edit", "delete"]
    },
    "detail": {
      "label": "详情",
      "type": "detail",
      "columns": ["id", "name", "created_at", "created_by"]
    }
  }
}
```

**字段类型映射**：

| JSON type | 数据库类型 | 前端组件 |
|-----------|----------|---------|
| `string` | VARCHAR(255) | 文本输入 |
| `text` | TEXT | 多行文本 |
| `number` | INT/BIGINT | 数字输入 |
| `decimal` | DECIMAL | 数字输入（带小数） |
| `datetime` | DATETIME | 日期时间选择 |
| `date` | DATE | 日期选择 |
| `boolean` | TINYINT(1) | 开关 |
| `enum` | VARCHAR(50) | 下拉选择 |
| `reference` | 外键 | 关联选择 |

### 4. 生成 Groovy 钩子文件

按以下模板生成 `<ModelName>.groovy`：

```groovy
/**
 * {{ModelName}} 模型业务逻辑
 *
 * 包含钩子方法：
 * - validate: 业务校验
 * - doBehavior: 核心业务逻辑
 * - updator: 字段联动
 * - onChange: 字段变更回调
 */

import com.qinqinxiaobao.report.uniplat.executor.*
import com.qinqinxiaobao.report.uniplat.engine.DO.*

class {{ModelName}} {

    /**
     * 业务校验
     * 返回 null 表示通过，返回字符串表示错误信息
     */
    static String validate(ActionValidatorContext ctx) {
        DataObject row = ctx.row
        PureDataObject inputs = ctx.inputs
        EnvDataObject env = ctx.env

        // TODO: 添加校验逻辑
        // 示例：
        // if (!inputs.getString("name")) {
        //     return "名称不能为空"
        // }

        return null
    }

    /**
     * 核心业务逻辑
     * 返回 null 走平台默认持久化
     * 返回 BehaviorResult 自定义结果
     */
    static Object doBehavior(ActionBehaviorContext ctx) {
        DataObject row = ctx.row
        PureDataObject inputs = ctx.inputs
        EnvDataObject env = ctx.env

        // TODO: 添加业务逻辑
        // 示例：
        // if (ctx.isInsert) {
        //     row.put("created_by", env.getUserId())
        // }

        return null
    }

    /**
     * 字段联动（主表）
     * 首次加载 sender 为空字符串，字段变更时 sender 为字段名
     */
    static void updator(ParameterUpdateMasterContext ctx) {
        EnvDataObject env = ctx.getEnv()
        Map variables = ctx.getVariables()
        String sender = ctx.getSender()

        // TODO: 添加联动逻辑
        // 示例：
        // if (sender == "" || sender == "type_id") {
        //     def options = host.getDataModel("Type").findAll().collect {
        //         [value: it.getKeyValue(), label: it.getString("name")]
        //     }
        //     ctx.setOptions("type_id", options)
        // }
    }

    /**
     * 字段变更回调
     */
    static void onChange(ParameterChangeContext ctx) {
        DataObject row = ctx.row
        String fieldName = ctx.fieldName
        Object newValue = ctx.newValue
        Map variables = ctx.variables

        // TODO: 添加 onChange 逻辑
    }
}
```

### 5. 向用户确认

生成文件后，向用户展示：
- 创建的文件列表
- JSON 中定义的字段和 Action
- Groovy 中的钩子方法说明

用 AskUserQuestion 询问是否需要调整字段、添加更多 Action 等。

### 6. 后续步骤建议

提示用户：
- 如需添加更多 Action → 使用 `/uniplat:create-action`
- 如需创建自定义 API → 使用 `/uniplat:create-service`
- 如需添加字段联动 → 编辑 Groovy 中的 `updator` 方法

---

## 注意事项

1. **模型名必须大驼峰**：`Order`、`PartyMember`，不能是 `order` 或 `party_member`
2. **表名自动推断**：`Order` → `t_order`，`PartyMember` → `t_party_member`
3. **JSON 中的 actions.parameters 是数组**：每个参数必须有 `name`、`label`、`type`
4. **Groovy 类名必须与文件名一致**：`Order.groovy` → `class Order`
5. **所有钩子方法都是 static**：Uniplat 的 Groovy 钩子都是静态方法
