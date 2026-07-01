---
name: coding-guide
description: Uniplat 低代码平台编码规范速查。在 Uniplat 项目中进行任何开发任务时参考。
version: 1.0.1
---

# Uniplat 编码规范速查

> 本 skill 是 Uniplat 低代码平台（Java + Groovy + JSON）的编码规范。任何 Uniplat 项目的编码任务都应遵循本文规范。

## 平台概览

```
JSON 配置（模型元数据、字段、Action、列表/详情视图）
    +
Groovy 脚本（Validator/Behavior/Updator/Service）
    =
可运行的业务系统
```

**核心思想**：Java 核心代码不写业务逻辑，所有业务通过 Groovy 钩子实现。

---

## 1. 项目结构

### 1.1 工作区与子项目

一个 Uniplat 工作区（Git 仓库或 workspace）包含多个**子项目**：

```
v3-refrom/                          # 工作区根目录
├── uniplat-main/                   # 平台主模块（引擎 + 配置 + schemas）
│   └── config/
│       └── schemas/                # JSON Schema 定义（开发参考）
├── uniplat_base/                   # 基础领域子项目
├── uniplat_common/                 # 公共领域子项目
├── hro/                            # 业务工作区（含多个子项目）
│   ├── bank_account/               # 子项目（独立 Git）
│   ├── hro_spview/                 # 子项目（独立 Git）
│   └── employment_project/         # 子项目
└── ...其他业务模块
```

### 1.2 子项目目录结构

每个子项目是一个独立的 Git 仓库，结构如下：

```
subproject_name/
├── .git/                           # 独立 Git 仓库
├── Entrances.groovy                # 入口菜单配置
├── consts/                         # 常量定义
│   ├── base/common.json
│   └── common.json
├── models/                         # 数据模型定义（核心）
│   ├── system/                     # 模型分组（子目录）
│   │   ├── system_user.json
│   │   └── system_user.groovy
│   └── Chat/                       # 另一分组
│       ├── UniplatChat.json
│       └── UniplatChat.groovy
├── services/                       # 领域服务（自定义 API）
│   ├── passport_api.groovy
│   └── client_api.groovy
├── monitors/                       # 监控/调度任务
├── dashboard/                      # 看板
├── process/                        # 工作流
└── test/                           # 测试
    └── models/
```

**命名规则**：
- 子项目名：小写 + 下划线（`hro_spview`、`uniplat_base`）
- 模型分组目录：CamelCase 或 snake_case（`system`、`Chat`、`business/order`）
- 模型文件名：与表名一致（snake_case 如 `system_user` 或 CamelCase 如 `UniplatChat`）
- 服务文件名：描述性 + `_api` 后缀（`passport_api`、`client_api`）

---

## 2. 模型 JSON 格式

每个模型对应一个 JSON 文件，格式如下：

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
        {"key": "1", "value": "正常"},
        {"key": "-1", "value": "已关闭"}
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
        "nullable": false
      }
    },
    {
      "property": "status",
      "label": "状态",
      "type": "mapping",
      "mapping": "status_mapping",
      "column": {
        "type": "INT",
        "defaultValue": "0"
      }
    },
    {
      "property": "city_id",
      "label": "城市",
      "type": "mapping",
      "mapping": "city_mapping"
    },
    {
      "property": "created_time",
      "label": "创建时间",
      "type": "date",
      "format": "yyyy-MM-dd HH:mm:ss"
    }
  ],

  "joint_defs": [],
  "calculator_defs": [],
  "references": [
    {
      "name": "owner",
      "model": "system_user",
      "relations": [
        {"referProperty": "uid", "referredProperty": "id"}
      ]
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
          {"property": "create_by", "result": "env.user_id.value"},
          {"property": "create_time", "result": "env.now.value"}
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
      "forward": "refresh"
    },
    {
      "name": "update",
      "when": "1",
      "label": "修改",
      "container": "dialog",
      "parameters": {
        "server": [
          {"property": "update_by", "result": "env.user_id.value"},
          {"property": "update_time", "result": "env.now.value"}
        ],
        "inputs": [
          {
            "property": "name",
            "label": "名称",
            "type": "text",
            "default_value": "object.name.value"
          }
        ]
      },
      "validator": "update_validator",
      "behavior": "updateBehavior",
      "forward": "refresh"
    }
  ],

  "list": {
    "label": "列表标题",
    "filters": [
      {"label": "名称", "field": "name", "type": "text"},
      {"label": "状态", "field": "status", "type": "enum", "mapping": "status_mapping"}
    ],
    "actions": ["insert"],
    "row_actions": ["update", "delete"],
    "field_groups": [
      {"label": "名称", "template": "{name}"},
      {"label": "状态", "template": "{status}"}
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

  "indexes": [
    {
      "name": "idx_status",
      "columns": ["status"]
    }
  ],

  "modelVersion": false
}
```

### 2.1 字段类型速查

| JSON type | 用途 | 说明 |
|-----------|------|------|
| `number` | 数字 | `format: "0"` 整数, `"0.00"` 两位小数 |
| `text` | 文本 | 单行文本输入 |
| `mapping` | 映射/枚举 | 需指定 `mapping` 字段引用 mapping_defs |
| `date` | 日期 | `format: "yyyy-MM-dd"` |
| `datetime` | 日期时间 | `format: "yyyy-MM-dd HH:mm:ss"` |
| `boolean` | 布尔 | 开关组件 |
| `hidden` | 隐藏 | 不显示在表单 |
| `file` | 文件 | 文件上传 |

### 2.2 Action 参数说明

**parameters.server**：服务端自动填充的参数（表达式）
- `env.user_id.value` — 当前用户 ID
- `env.now.value` — 当前时间
- `env.xid.value` — 当前组织 ID
- 任意 Groovy 表达式

**parameters.inputs**：用户填写的表单参数
- `property` — 字段名
- `label` — 标签
- `type` — 类型（text/number/mapping/date/hidden...）
- `required` — 是否必填
- `default_value` — 默认值表达式（如 `object.name.value`）
- `mapping` — 当 type=mapping 时引用的映射名
- `updator` — 字段联动方法名
- `span` — 表单列宽（12=半行，24=整行）
- `is_param` — 是否作为 URL 参数传递

**Action 属性**：
- `name` — Action 唯一标识
- `when` — 显示条件表达式（`"1"` 总是显示）
- `label` — 按钮文字
- `container` — `"dialog"` 弹窗 / `"none"` 直接执行
- `behavior` — 行为方法名（对应 Groovy 方法）
- `validator` — 校验方法名
- `forward` — 执行后跳转（`"refresh"` 刷新 / `"close"` 关闭）
- `tx` — 是否开启事务
- `on` — `"object"` 表示在数据行上操作
- `prompt` — 执行前提示文字

---

## 3. 模型 Groovy 格式

每个模型对应一个 Groovy 文件，格式如下：

```groovy
package models.system

import com.qinqinxiaobao.report.uniplat.engine.DO.DataObject
import com.qinqinxiaobao.report.uniplat.engine.DO.EnvDataObject
import com.qinqinxiaobao.report.uniplat.engine.DO.PureDataObject
import com.qinqinxiaobao.report.uniplat.executor.ActionBehaviorContext
import com.qinqinxiaobao.report.uniplat.executor.ActionValidatorContext
import com.qinqinxiaobao.report.uniplat.executor.ParameterUpdateMasterContext
import com.qinqinxiaobao.report.uniplat.executor.ParameterChangeContext
import com.qinqinxiaobao.report.uniplat.executor.BehaviorResult
import com.qinqinxiaobao.report.uniplat.host.Host
import com.qinqinxiaobao.report.utils.AssertUtils

class model_name {
    def model_name = "model_name"

    // ====== Validator ======
    def update_validator(ActionValidatorContext ctx) {
        def obj = ctx.dataList.fetchOne()
        def inputs = ctx.inputs
        // 使用 AssertUtils 抛出异常（校验失败）
        AssertUtils.isTrue(inputs.getString("name"), "名称不能为空")
        AssertUtils.isFalse(obj.getBoolean("is_default"), "默认项不能修改")
    }

    // ====== Behavior ======
    def insertBehavior(ActionBehaviorContext ctx) {
        def inputs = ctx.inputs
        def host = ctx.host
        def now = host.getTimestamp()
        def operator = host.getUser().id as long

        def id = ctx.dataModel.insertByMap([
            name       : inputs.name.value,
            status     : inputs.status.value as int,
            create_by  : operator,
            create_time: now,
            update_by  : operator,
            update_time: now,
            is_del     : 0
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
        def now = ctx.host.getTimestamp()
        def operator = ctx.host.getUser().id as long

        obj.update([
            name       : inputs.name.value,
            update_time: now,
            update_by  : operator
        ])

        return [
            result: 0,
            id    : obj.getKeyValue(),
            msg   : "修改成功"
        ]
    }

    // ====== Updator（字段联动） ======
    def status_updator(ParameterUpdateMasterContext context) {
        if (!context.sender || context.sender == "type") {
            return [default_value: "1"]
        }
    }

    def name_updator(ParameterUpdateMasterContext context) {
        if (!context.sender || context.sender == "city_id") {
            // 根据城市设置默认名称
            def cityId = context.variables.city_id
            if (cityId) {
                return [default_value: "名称_${cityId}"]
            }
        }
    }

    // ====== onChange（字段变更回调） ======
    def onTypeChange(ParameterChangeContext context) {
        def type = context.params.type as String
        if (type == "CORP") {
            return [
                variables: [show_tax_field: true]
            ]
        }
    }

    // ====== 自定义方法（被 action 的 behavior 引用） ======
    def customAction(ActionBehaviorContext context) {
        def obj = context.dataList.fetchOne()
        // 业务逻辑...
        return [result: 0, msg: "操作成功"]
    }

    // ====== 自定义工具方法（被 invokeModelFunc 调用） ======
    def getDisplayInfo(DataObject obj) {
        return "${obj.name.value} - ${obj.status.display}"
    }
}
```

### 3.1 关键约定

| 项目 | 规范 |
|------|------|
| 方法类型 | **实例方法**（不是 static） |
| 返回类型 | 使用 `def`（不显式声明） |
| Validator 校验 | 用 `AssertUtils.isTrue/isFalse` 抛异常，**不返回错误字符串** |
| Behavior 返回 | `[result: 0, id: ..., msg: "..."]` |
| Updator 返回 | `[default_value: ..., readonly: ..., options: ...]` |
| onChange 返回 | `[variables: [...]]` 更新变量 |
| package 声明 | `package models.分组目录名` |
| model_name 属性 | 类中必须声明 `def model_name = "model_name"` |

### 3.2 Context 类速查

| 类 | 用途 | 关键字段 |
|----|------|---------|
| `ActionValidatorContext` | 校验 | `ctx.dataList`, `ctx.inputs`, `ctx.env`, `ctx.host`, `ctx.dataModel` |
| `ActionBehaviorContext` | 行为 | `ctx.dataList`, `ctx.inputs`, `ctx.env`, `ctx.host`, `ctx.dataModel` |
| `ParameterUpdateMasterContext` | Updator | `context.sender`, `context.variables`, `context.params`, `context.env` |
| `ParameterChangeContext` | onChange | `context.params`, `context.host`, `context.variables` |

### 3.3 DataModel 常用方法

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
obj.update()  // 使用 obj 当前值

// 删除
obj.delete()

// 数据访问
def value = obj.name.value        // 字段值
def display = obj.status.display  // 映射显示值
def keyVal = obj.getKeyValue()    // 主键值
def dict = obj.toDict(env)        // 转为 Map

// 关联数据
def related = obj.get("refName.fieldName").value
```

### 3.4 Host 常用方法

```groovy
def host = Host.getInstance()  // 或 ctx.host

// 获取模型
def model = host.getDataModel("system_user")

// 获取用户信息
def user = host.getUser()
def userId = user.id
def userName = user.name

// 获取时间
def now = host.getTimestamp()  // 字符串格式时间

// 调用模型方法
def result = host.invokeModelFunc("model_name", "method_name", arg1, arg2)

// 发送事件
host.sendModelEvent(modelName, action, row)
```

---

## 4. 领域服务（Service）

### 4.1 文件格式

```groovy
package services

import com.qinqinxiaobao.report.uniplat.gateway.GatewayContext
import com.qinqinxiaobao.report.uniplat.host.Host

class service_name_api {

    def methodName(GatewayContext ctx) {
        def host = Host.getInstance()
        // 获取参数
        def param = ctx.getBody().get("paramName")
        def queryParam = ctx.parameters["key"]?.getAt(0)

        // 业务逻辑
        def model = host.getDataModel("system_user")
        def users = model.queryDataList([status: 1])

        return users.list.collect {
            [id: it.getKeyValue(), name: it.name.value]
        }
    }
}
```

### 4.2 服务 API 端点

```
POST /general/project/{subproject}/service/{ServiceClass}/{method}
```

示例：
```
POST /general/project/hro_spview/service/passport_api/getUsersInfo
Body: {"userMemberIds": "1,2,3"}
```

### 4.3 GatewayContext 字段

```groovy
ctx.getBody()           // Map 请求体
ctx.getBody().get("key")// 获取 body 参数
ctx.parameters          // Map<String, String[]> 查询参数
ctx.parameters["key"]?.getAt(0)  // 取第一个值
ctx.request             // HttpServletRequest
ctx.host                // Host 实例（可选，也可 Host.getInstance()）
```

---

## 5. 常用模式

### 5.1 插入数据并设为默认

```groovy
def insertBehavior(ActionBehaviorContext ctx) {
    def inputs = ctx.inputs
    def operator = ctx.host.getUser().id as long
    def now = ctx.host.getTimestamp()

    // 如果设为默认，先取消其他默认
    if (inputs.is_default.value) {
        ctx.dataModel.queryDataList([uid: operator, is_default: 1]).asStream().forEach {
            it.update([is_default: 0, update_time: now, update_by: operator])
        }
    }

    def id = ctx.dataModel.insertByMap([
        uid        : operator,
        is_default : inputs.is_default.value ? 1 : 0,
        create_time: now,
        create_by  : operator
    ]).id

    return [result: 0, id: id, msg: "添加成功"]
}
```

### 5.2 事务处理

```groovy
import com.qinqinxiaobao.report.db.DataSourceFactory
import com.qinqinxiaobao.report.db.DbConsts

DataSourceFactory.transaction(DbConsts.HOST, {
    // 事务内的操作
    obj1.update()
    obj2.delete()
} as Runnable)
```

### 5.3 直接 SQL 查询

```groovy
import com.qinqinxiaobao.report.db.DataSourceFactory
import com.qinqinxiaobao.report.db.DbConsts

def ds = DataSourceFactory.getDataSource(DbConsts.HOST)
def results = ds.queryForList("SELECT * FROM table WHERE status = ?", 1)
```

### 5.4 调用其他模型的方法

```groovy
def host = ctx.host
// 调用 model Groovy 中的方法
def result = host.invokeModelFunc("system_user", "getUserDisplay", userId)
```

---

## 6. Entrances.groovy

```groovy
class UniplatEntrances {
    def SUB_PROJECT_TITLE = '子项目显示名'
    def order = 1
    def entrancesV2 = []
}
```

---

## 7. 常量定义

`consts/common.json` 或 `consts/base/common.json`：

```json
{
  "mappings": {
    "user_type_mapping": {
      "name": "user_type_mapping",
      "mapping_values": [
        {"key": "1", "value": "平台用户"},
        {"key": "2", "value": "匿名用户"}
      ]
    }
  },
  "joints": {}
}
```

---

## 8. 开发规范

1. **JSON 文件用 2 空格缩进**，Groovy 文件用 **4 空格缩进**
2. **Groovy 类名与文件名一致**（`system_user.groovy` → `class system_user`）
3. **package 声明对应分组目录**（`models/system/system_user.groovy` → `package models.system`）
4. **所有业务方法都是实例方法**（不是 static）
5. **使用 `def` 声明方法和变量**
6. **Validator 用 AssertUtils 抛异常**，不要返回字符串
7. **Behavior 返回 `[result: 0, id: ..., msg: "..."]`**
8. **Updator 返回 Map**：`[default_value: ..., readonly: ..., options: ...]`
9. **数据访问用 `obj.field.value`**，不直接 `obj.get("field")`
10. **server 参数用表达式**：`env.user_id.value`、`env.now.value`
11. **mapping_refs 引用其他子项目的常量**：通过 `subproject` + `filename` + `key` 路径

---

## 9. 开发前检查清单

开始编码前：
- [ ] 确认子项目路径和模型分组目录
- [ ] 确认数据库表名和数据源（`database` 字段）
- [ ] 参考 `config/schemas/datamodel.json` 了解 JSON Schema
- [ ] 查找相似模型参考现有写法

---

## 10. 关键文件位置

| 内容 | 位置 |
|------|------|
| JSON Schema | `uniplat-main/config/schemas/datamodel.json` |
| 基础模型 | `uniplat_base/models/` |
| 公共模型 | `uniplat_common/models/` |
| 业务模型 | `hro/{子项目}/models/` |
| 领域服务 | `hro/{子项目}/services/` |
| 常量定义 | `{子项目}/consts/` |
