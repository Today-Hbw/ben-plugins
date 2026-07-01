---
name: coding-guide
description: Uniplat 低代码平台编码规范速查。在 Uniplat 项目中进行任何开发任务时参考。
version: 1.0.2
---

# Uniplat 编码规范速查

> 本 skill 是 Uniplat 低代码平台（Java + Groovy + JSON）的编码规范。任何 Uniplat 项目的编码任务都应遵循本文规范。

## 平台概览

```
JSON 配置（模型元数据、字段、Action、列表/详情视图）
    +
Groovy 脚本（Validator/Behavior/Updator/Service/EventHandler）
    =
可运行的业务系统
```

**核心思想**：Java 核心代码不写业务逻辑，所有业务通过 Groovy 钩子实现。

---

## 1. 项目结构

### 1.1 工作区与子项目

一个 Uniplat 工作区包含多个**子项目**，每个子项目是独立 Git 仓库：

```
v3-refrom/                          # 工作区根目录
├── uniplat-main/                   # 平台主模块（引擎 + 配置 + schemas）
│   └── config/schemas/             # JSON Schema 定义（开发参考）
├── uniplat_base/                   # 基础领域子项目
├── uniplat_common/                 # 公共领域子项目
├── hro/                            # 业务工作区（含多个子项目）
│   ├── bank_account/               # 子项目（独立 Git）
│   ├── hro_spview/                 # 子项目（独立 Git）
│   └── ...
```

### 1.2 子项目目录结构

```
subproject_name/
├── .git/                           # 独立 Git 仓库
├── Entrances.groovy                # 入口菜单配置
├── consts/                         # 常量定义（mapping、joint 等）
│   ├── base/common.json
│   └── common.json
├── models/                         # 数据模型定义（核心）
│   ├── system/                     # 模型分组（子目录）
│   │   ├── system_user.json
│   │   └── system_user.groovy
│   └── business/order/
├── services/                       # 领域服务（自定义 API）
│   └── passport_api.groovy
├── monitors/                       # 监控/调度任务
├── dashboard/                      # 看板
└── process/                        # 工作流
```

**注意**：`test/` 目录是**废弃机制**，新项目不创建。

---

## 2. 模型 JSON 格式

```json
{
  "database": "(host)",
  "table": "system_user",
  "modelDescription": "用户",
  "key_field": "id",
  "data_right": true,

  "mapping_defs": [
    {
      "name": "status_mapping",
      "mapping_values": [
        {"key": "0", "value": "待审核"},
        {"key": "1", "value": "正常"}
      ]
    },
    {
      "name": "city_mapping",
      "database": "(host)",
      "sql": "SELECT id, name FROM uniplat_district WHERE is_del = 0"
    }
  ],

  "mapping_refs": [
    {
      "subproject": "uniplat_base",
      "filename": "base/common.json",
      "key": "$.mappings.uniplat_user_type_mapping"
    }
  ],

  "field_defs": [
    {
      "property": "id",
      "label": "id",
      "type": "number",
      "format": "0",
      "column": {"type": "BIGINT", "autoIncrement": true, "comment": "主键"}
    },
    {
      "property": "status",
      "label": "状态",
      "type": "mapping",
      "mapping": "status_mapping",
      "column": {"type": "INT", "defaultValue": "0"}
    }
  ],

  "joint_defs": [
    {
      "name": "member_role",
      "sql": "SELECT xm.id mid, group_concat(role.name) role_names FROM ...",
      "key_field": "mid",
      "joint_field": "mid",
      "field_defs": [
        {"property": "mid", "label": "成员"},
        {"property": "role_names", "label": "角色名"}
      ]
    }
  ],

  "calculator_defs": [],
  "references": [
    {
      "name": "owner",
      "model": "system_user",
      "relations": [{"referProperty": "uid", "referredProperty": "id"}]
    }
  ],

  "action_defs": [
    {
      "name": "insert",
      "when": "1",
      "label": "新增",
      "container": "dialog",
      "parameters": {
        "server": [
          {"property": "created_by", "result": "env.user_id.value"},
          {"property": "created_time", "result": "env.now.value"}
        ],
        "inputs": [
          {"property": "name", "label": "名称", "type": "text", "required": true, "span": 12}
        ]
      },
      "behavior": "insert",
      "forward": "refresh"
    }
  ],

  "dataFilters": [
    {"field": "is_del", "value": "0"}
  ],

  "list": {
    "label": "用户列表",
    "filters": [
      {"label": "状态", "field": "status", "type": "enum", "mapping": "status_mapping"}
    ],
    "actions": ["insert"],
    "row_actions": ["update", "delete"],
    "field_groups": [
      {"label": "名称", "template": "{name}"},
      {"label": "状态", "template": "{status}"}
    ],
    "dataFilters": [
      {"field": "status", "value": "1"}
    ],
    "detail_action_visible": false
  },

  "detail": {
    "title_template": "",
    "actions": [],
    "header": {"field_groups": [], "actions": []},
    "pages": []
  },

  "modelVersion": false
}
```

### 2.1 behavior 字段的关键约定

| behavior 值 | 含义 | 是否需要 Groovy 方法 |
|-------------|------|---------------------|
| `"insert"` | 平台内置插入 | **不需要** |
| `"update"` | 平台内置更新 | **不需要** |
| `"delete"` | 平台内置删除 | **不需要** |
| `"myCustomMethod"` | 自定义方法 | **需要** |
| `""` | 无行为（纯跳转） | **不需要** |

**默认情况下，insert/update/delete 使用平台内置 behavior，不需要在 Groovy 中写方法。** 只有需要自定义业务逻辑时才改为自定义方法名并写 Groovy。

### 2.2 when 条件空值安全

当 `when` 条件使用 `object.field.value` 时，**必须考虑字段可能为空**：

```json
// 安全写法（使用 ?. 安全导航）
{"when": "object.status?.value as int == 0"}
{"when": "object.field?.value"}
{"when": "object.status?.value && object.type?.value == 'CORP'"}
{"when": "!object.is_del?.value"}

// 确定有值的字段可以不用 ?.
{"when": "1"}
```

**规则**：凡是可能为空的字段（特别是关联字段、可选字段），都用 `?.value`。

### 2.3 dataFilters 说明

`dataFilters` 用于自动过滤数据，可出现在：

| 位置 | 作用范围 |
|------|---------|
| 模型顶层 `dataFilters` | 对所有查询生效 |
| `list.dataFilters` | 仅对列表查询生效 |
| `list.pages[].dataFilters` | 对特定页签生效 |

常用模式：
```json
{"field": "is_del", "value": "0"}              // 软删除过滤
{"field": "xid", "value": "context.xid"}       // 组织隔离
{"field": "valid", "value": "1"}               // 有效数据过滤
{"field": "poid", "value": "org.passports.PASSPORT"}  // 动态值
```

### 2.4 虚拟模型（SQL 模型）

模型可以不绑定实体表，而是用 SQL 定义视图：

```json
{
  "table": "virtual_view_name",
  "sql": "SELECT id, name, COUNT(*) as count FROM real_table GROUP BY id",
  "database": "(host)",
  "key_field": "id",
  "field_defs": [...]
}
```

虚拟模型只支持查询，不支持 insert/update/delete。

### 2.5 joint_defs 关联定义

`joint_defs` 用于 SQL JOIN 关联，将多表数据合并到一个模型：

```json
"joint_defs": [
  {
    "name": "member_role",
    "sql": "SELECT member_id mid, GROUP_CONCAT(role_name) roles FROM ... GROUP BY member_id",
    "key_field": "mid",
    "joint_field": "id",
    "field_defs": [
      {"property": "mid", "label": "成员ID"},
      {"property": "roles", "label": "角色列表"}
    ]
  }
]
```

访问 joint 字段：`obj.get("member_role.roles").value`

---

## 3. 模型 Groovy 格式

### 3.1 基本结构

```groovy
package models.system

import com.qinqinxiaobao.report.uniplat.engine.DO.DataObject
import com.qinqinxiaobao.report.uniplat.executor.ActionBehaviorContext
import com.qinqinxiaobao.report.uniplat.executor.ActionValidatorContext
import com.qinqinxiaobao.report.uniplat.executor.ParameterUpdateMasterContext
import com.qinqinxiaobao.report.uniplat.executor.ParameterChangeContext
import com.qinqinxiaobao.report.uniplat.models.event.ModelEventContext
import com.qinqinxiaobao.report.uniplat.host.Host
import com.qinqinxiaobao.report.utils.AssertUtils

class model_name {
    def model_name = "model_name"

    // ====== 以下内容都是可选的，按需添加 ======

    // Validator：仅当 action 配置了 "validator": "xxx" 时需要
    def my_validator(ActionValidatorContext ctx) { ... }

    // Behavior：仅当 action 的 behavior 不是 "insert"/"update"/"delete" 时需要
    def my_behavior(ActionBehaviorContext ctx) { ... }

    // Updator：仅当 input 配置了 "updator": "xxx" 时需要
    def my_updator(ParameterUpdateMasterContext ctx) { ... }

    // onChange：仅当 input 配置了 "on_change": "xxx" 时需要
    def my_onChange(ParameterChangeContext ctx) { ... }

    // 事件处理：仅当 JSON 中配置了 eventSubs 时需要
    def my_event_handler(ModelEventContext ctx) { ... }

    // 自定义方法：被 invokeModelFunc 调用
    def getDisplayInfo(DataObject obj) { ... }
}
```

### 3.2 关键约定

| 项目 | 规范 |
|------|------|
| 方法类型 | **实例方法**（不是 static） |
| 返回类型 | 使用 `def` |
| Validator | 用 `AssertUtils` 抛异常，不返回错误字符串 |
| Behavior | 返回 `[result: 0, id: ..., msg: "..."]` |
| Updator | 返回 `[default_value: ..., readonly: ..., options: ...]` |
| package | 对应分组目录：`package models.system` |
| model_name | 类中必须声明 `def model_name = "model_name"` |

### 3.3 当行为使用平台内置时

如果 JSON 中 `behavior: "insert"`、`behavior: "update"`，**Groovy 中不需要写对应方法**。
只有以下情况才需要写 Groovy 方法：
- action 配置了 `validator`
- action 的 `behavior` 是自定义方法名
- input 配置了 `updator`
- input 配置了 `on_change`
- 需要处理模型事件

### 3.4 Context 类速查

| 类 | 用途 | 关键字段 |
|----|------|---------|
| `ActionValidatorContext` | 校验 | `ctx.dataList`, `ctx.inputs`, `ctx.env`, `ctx.host`, `ctx.dataModel` |
| `ActionBehaviorContext` | 行为 | `ctx.dataList`, `ctx.inputs`, `ctx.env`, `ctx.host`, `ctx.dataModel` |
| `ParameterUpdateMasterContext` | Updator | `context.sender`, `context.variables`, `context.params`, `context.env` |
| `ParameterChangeContext` | onChange | `context.params`, `context.host`, `context.variables` |
| `ModelEventContext` | 事件 | `ctx.host`, `ctx.message` (含 action, ids) |

### 3.5 DataModel 常用方法

```groovy
def dataModel = ctx.dataModel
def host = ctx.host

// 查询
def list = dataModel.queryDataList([status: 1, is_del: 0])
def one = dataModel.getByKeyField(id)
def count = list.count()
def first = list.fetchOne()
def all = list.list

// 插入
def newId = dataModel.insertByMap([name: "xxx", status: 1]).id

// 更新
obj.update([name: "new_name", status: 2])

// 删除
obj.delete()

// 数据访问
def value = obj.name.value        // 字段值
def display = obj.status.display  // 映射显示值
def keyVal = obj.getKeyValue()    // 主键值

// 关联数据（joint）
def related = obj.get("jointName.fieldName").value

// 引用关系（reference）
def ref = obj.referTo("refName").fieldName.value
```

---

## 4. 事件系统

### 4.1 ModelEventContext（模型事件处理）

在 Groovy 中处理模型数据变更事件：

```groovy
import com.qinqinxiaobao.report.uniplat.models.event.ModelEventContext

class model_name {
    def model_name = "model_name"

    def onModelChange(ModelEventContext ctx) {
        def msg = ctx.getMessage()
        def action = msg.getAction()      // "insert", "update", "delete", 或自定义
        def ids = msg.getIds()            // 受影响的数据 ID 列表
        def host = ctx.getHost()

        if (action == "insert") {
            def model = host.getDataModel("model_name")
            def obj = model.getByKeyField(ids[0])
            // 处理新增事件...
        }
    }
}
```

JSON 中配置事件订阅：

```json
"eventSubs": [
  {
    "action": "insert",
    "handler": "onModelChange"
  }
]
```

### 4.2 发送事件

```groovy
def host = Host.getInstance()
host.sendModelEvent("model_name", "custom_action", dataObject)
```

---

## 5. 领域服务（Service）

```groovy
package services

import com.qinqinxiaobao.report.uniplat.gateway.GatewayContext
import com.qinqinxiaobao.report.uniplat.host.Host

class service_name_api {

    def methodName(GatewayContext ctx) {
        def host = Host.getInstance()
        def body = ctx.getBody()
        def param = body.get("param")?.toString()

        def model = host.getDataModel("system_user")
        def list = model.queryDataList([status: 1])

        return list.list.collect { [id: it.getKeyValue(), name: it.name.value] }
    }
}
```

**API 端点**：
```
POST /general/project/{subproject}/service/{ServiceClass}/{method}
```

**GatewayContext 字段**：
- `ctx.getBody().get("key")` — body 参数
- `ctx.parameters["key"]?.getAt(0)` — URL 参数
- `ctx.request` — HttpServletRequest

---

## 6. 常用模式

### 6.1 软删除

JSON：
```json
{
  "name": "delete",
  "when": "!object.is_del?.value",
  "parameters": {
    "server": [
      {"property": "is_del", "result": "1"},
      {"property": "updated_time", "result": "env.now.value"}
    ],
    "inputs": []
  },
  "behavior": "update",
  "forward": "refresh"
}
```

list 中加过滤：
```json
"dataFilters": [{"field": "is_del", "value": "0"}]
```

### 6.2 事务处理

```groovy
import com.qinqinxiaobao.report.db.DataSourceFactory
import com.qinqinxiaobao.report.db.DbConsts

DataSourceFactory.transaction(DbConsts.HOST, {
    obj1.update()
    obj2.delete()
} as Runnable)
```

### 6.3 直接 SQL 查询

```groovy
import com.qinqinxiaobao.report.db.DataSourceFactory
import com.qinqinxiaobao.report.db.DbConsts

def ds = DataSourceFactory.getDataSource(DbConsts.HOST)
def results = ds.queryForList("SELECT * FROM table WHERE status = ?", 1)
```

### 6.4 Updator 返回格式

```groovy
def my_updator(ParameterUpdateMasterContext context) {
    if (!context.sender || context.sender == "type_id") {
        return [
            default_value: "1",
            readonly: false,
            visible: true,
            options: [[value: "1", label: "选项1"], [value: "2", label: "选项2"]]
        ]
    }
}
```

### 6.5 获取过滤条件（dataFilter 相关）

在 Groovy 中，可以通过 context 获取当前查询的过滤条件：

```groovy
def myBehavior(ActionBehaviorContext ctx) {
    // 获取当前列表的过滤参数
    def masterParams = ctx.masterParams  // 列表过滤参数

    // 获取环境信息
    def xid = ctx.env.xid.value  // 当前组织
    def userId = ctx.env.user_id.value
}
```

---

## 7. 开发规范

1. **不创建 test 目录**：这是废弃机制
2. **默认用平台内置 behavior**：insert/update/delete 不需要 Groovy 方法
3. **when 条件用 `?.value` 安全导航**
4. **dataFilters 用于自动过滤**：特别是 `is_del = 0`
5. **Groovy 类名与文件名一致**
6. **package 声明对应分组目录**
7. **所有业务方法都是实例方法**，用 `def` 声明
8. **Validator 用 AssertUtils 抛异常**
9. **Behavior 返回 `[result: 0, id: ..., msg: "..."]`**
10. **数据访问用 `obj.field.value`**

---

## 8. 关键文件位置

| 内容 | 位置 |
|------|------|
| JSON Schema | `uniplat-main/config/schemas/datamodel.json` |
| 基础模型 | `uniplat_base/models/` |
| 公共模型 | `uniplat_common/models/` |
| 业务模型 | `hro/{子项目}/models/` |
| 领域服务 | `hro/{子项目}/services/` |
| 常量定义 | `{子项目}/consts/` |

---

## 9. 开发前检查清单

- [ ] 确认子项目路径和模型分组目录
- [ ] 确认数据库表名和数据源
- [ ] 参考 `config/schemas/datamodel.json` 了解 JSON Schema
- [ ] 查找相似模型参考现有写法
- [ ] 确认是否需要自定义 behavior（默认用内置）
- [ ] 确认 when 条件的空值安全
- [ ] 确认 list 的 dataFilters 软删除过滤
