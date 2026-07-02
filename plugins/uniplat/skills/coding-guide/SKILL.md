---
name: coding-guide
description: Uniplat 低代码平台编码规范速查。在 Uniplat 项目中进行任何开发任务时参考。
version: 1.0.3
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

### 架构定位（背景，了解即可）

平台运行时由三个服务组成：**Gateway**（统一入口：认证/路由/限流）→ **Configurator**（配置中心：加载并版本化 JSON/Groovy 配置）→ **Runner**（运行引擎：执行业务与 Groovy 钩子）。开发者只写「JSON 模型 + Groovy 钩子」，由 Runner 加载执行。两个核心运行时对象：

- **Host**（单例，平台"大脑"）：`Host.getInstance()`，用它拿数据模型/用户/子项目等资源。
- **SubProject**（领域容器）：所有 Groovy 钩子的归属与分发中心（Validator/Behavior/Updator/Service/事件…都由它调用）。

> 本文只讲「怎么写模型/Action/服务」。三服务内部机制、配置加载、事件总线、调度等运维细节不在编码范围。

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

### 2.2 when 与 enable：显示 vs 启用，用 object 先判非空

action 上有两个条件表达式，**含义不同**：

| 字段 | 含义 | 不满足时 |
|------|------|---------|
| `when` | 该 action **是否显示** | 隐藏（看不见） |
| `enable`（= `enabled`） | 该 action **是否启用/可点击** | 置灰但仍显示，且不执行该逻辑 |

> `enable` 与 `enabled` 是**同一语义的两种拼写**：`action_defs[]` 里多写 `enable`，list 动作引用 / intent / JSON Schema 里写 `enabled`。二者可同时与 `when` 并列出现。

**空值安全铁律**：`when` / `enable` 表达式一旦引用 `object`，**必须先 `object != null &&` 守卫**。因为在 `on:"list"`/`on:"none"`（列表级、无选中行）等场景 `object` 本身就是 null，直接 `object.字段.value` 会抛 NPE。

```json
// ✅ 推荐：先守卫 object 非空，再判业务条件
{"enable": "object != null && object.getValue('Status') == 0"}
{"when":   "object != null && object.getValue('CurrentStatus') == 0"}
{"when":   "object != null && (object.Status.value==2 || object.Status.value==3)"}

// ✅ 常量条件无需守卫
{"when": "1"}
{"enable": "1"}

// ❌ 反面：object 可能为 null 时直接点属性 → NPE
{"enable": "object.Status.value == 0"}
```

**字段取值的两种写法**（都可用）：
- `object.getValue('字段')` —— 返回原始值，字段不存在返回 null，`== 值` 比较不 NPE（推荐用于条件）；
- `object.字段.value` —— 直接属性访问，字段包装为 null 时会 NPE，个别可空字段可叠加 `object.字段?.value`。

**关联/joint 字段**用 `object.get('本地键#joint名.字段').value`（`#` 分隔本地键与 `joint名.字段`），例如：
```json
{"when": "object != null && object.get('id#balance_refund_order_record_joint.OrderId').value != ''"}
```

> 说明：参考项目里大量历史表达式直接写 `object.字段.value` 无守卫——那是应被纠正的隐患，新写法一律先判 `object != null`。

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
// package 声明可选：参考项目里有的模型 groovy 声明了 package（对应分组目录），有的不声明也能跑。
// 建议按分组目录声明以便组织，如 package models.system。
package models.system

import com.qinqinxiaobao.report.uniplat.engine.DO.DataObject
import com.qinqinxiaobao.report.uniplat.executor.ActionBehaviorContext
import com.qinqinxiaobao.report.uniplat.executor.ActionValidatorContext
import com.qinqinxiaobao.report.uniplat.executor.BehaviorResult          // 自定义 behavior 返回类型（常用）
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
| 方法类型 | **实例方法**（不是 static），用 `def` 声明 |
| 返回类型 | 使用 `def` |
| Validator | 用 `AssertUtils` 抛异常（也可 `throw new ValidateException("...")`），不返回错误字符串 |
| Behavior | 返回 `BehaviorResult`（推荐，见 3.2.1）或字典 `[result: 0, id: ..., msg: "..."]` |
| Updator | 返回 `[default_value: ..., readonly: ..., options: ...]` |
| package | **可选**（建议按分组目录声明，如 `package models.system`；不声明也能运行） |
| model_name | 类中声明 `def model_name = "model_name"` |

### 3.2.1 Behavior 返回值：BehaviorResult 或字典

两种都被平台接受，参考项目里 `BehaviorResult` 更常用：

```groovy
import com.qinqinxiaobao.report.uniplat.executor.BehaviorResult

// 方式 A：BehaviorResult 类（推荐）
def approve_behavior(ActionBehaviorContext ctx) {
    def id = ctx.dataList.fetchOne().getKeyValue()
    BehaviorResult result = new BehaviorResult(0, "审批成功", id)  // (code, msg, id)
    result.data = [modelName: model_name]                          // 可选附加数据
    return result
}

// 方式 B：字典（等价，简单场景）
def publish_behavior(ActionBehaviorContext ctx) {
    // ...
    return [result: 0, id: id, msg: "发布成功"]
}
```

### 3.3 当行为使用平台内置时

如果 JSON 中 `behavior: "insert"`、`behavior: "update"`，**Groovy 中不需要写对应方法**。
只有以下情况才需要写 Groovy 方法：
- action 配置了 `validator`
- action 的 `behavior` 是自定义方法名
- input 配置了 `updator`
- input 配置了 `on_change`
- 需要处理模型事件

### 3.4 Context 类速查

> 属性均可用**属性式或方法式**访问：`ctx.sender` ≡ `ctx.getSender()`、`ctx.params` ≡ `ctx.getParams()`、`ctx.inputs` ≡ `ctx.getInputs()`、`ctx.host` ≡ `ctx.getHost()` 等。平台按 Groovy 方法**参数类型**自动匹配对应 Context，写对参数类型即可。

| 类（包） | 用途 | 关键字段/方法 |
|----------|------|--------------|
| `ActionValidatorContext`（`uniplat.executor`） | 校验 | `ctx.dataList`、`ctx.inputs`、`ctx.env`、`ctx.host`、`ctx.dataModel`、`ctx.variables` |
| `ActionBehaviorContext`（`uniplat.executor`） | 行为 | `ctx.dataList`、`ctx.inputs`、`ctx.env`、`ctx.host`、`ctx.dataModel`、`ctx.variables` |
| `ActionCustomInitContext`（`uniplat.executor`） | `customInitFunc` 动态生成 ActionDef | `ctx.dataList`、`ctx.env`、`ctx.host` |
| `ParameterUpdateMasterContext`（`uniplat.executor`） | 主表 Updator（字段联动） | `ctx.sender`、`ctx.params`、`ctx.variables`、`ctx.env`、`ctx.getParameter()` |
| `ParameterUpdateDetailContext`（`uniplat.executor`） | 详情行 Updator | 同上，附详情行上下文 |
| `ParameterChangeContext`（`uniplat.executor`） | onChange | `ctx.params`、`ctx.host`、`ctx.variables` |
| `ModelEventContext`（`uniplat.models.event`） | 模型事件 | `ctx.getMessage()`（`.getAction()/.getIds()/.getModelName()`）；取 Host：`ctx.getHost()` 或 `Host.getInstance()` 均可 |
| `ActionPageValuesFuncContext`（`uniplat.host`） | `page_values_func` 页面值计算 | `ctx.masterParams.get("字段")`（读列表预过滤参数）、`ctx.dataModel`、`ctx.env` |
| `FilterUpdateContext`（`uniplat.host`） | 列表筛选联动 updator | `ctx.sender`、`ctx.dataModel`、`ctx.getFilterValues().get("字段")` 或 `ctx.filterValues.getAt("字段")`（读当前筛选值） |
| `GatewayContext` / `AnonymousGatewayContext`（`uniplat.gateway`） | 领域服务（标准/匿名） | `ctx.host`、`ctx.body`、`ctx.parameters`、`ctx.request`、`ctx.pathValue`（详见 create-service） |
| `StreamApiContext` / `AnonymousStreamApiContext`（`uniplat.gateway`） | SSE 流式服务 | 同 GatewayContext + 流式输出 |
| `ModelServiceContext` / `AnonymousModelServiceContext`（`uniplat.models.service`） | 模型服务（绑定模型的自定义 API） | `ctx.dataModel`、`ctx.host`、请求参数 |
| `MappingCustomFuncContext`（`uniplat.models.mapping`） | `mapping.custom_func` 自定义映射 | `ctx.dataModel`、`ctx.env` |
| `JointCustomFuncContext`（`uniplat.models.joint`） | `joint.custom_function` 自定义关联 | `ctx.dataModel`、`ctx.env` |

### 3.5 内置方法速查（DataModel / DataObject / DataList）

> 以下方法名与签名**照引擎源码**（`models/meta/DataModel.java`、`engine/DO/DataObject.java`、`engine/DataList.java`）整理，为常用子集；完整 API 以源码为准。

**三者关系**：`DataModel`（模型，`host.getDataModel("x")` 得）→ 查询返回 `DataList`（列表）→ 取出 `DataObject`（单条行）。

#### DataModel（模型级操作）

| 类别 | 方法 | 说明 |
|------|------|------|
| 查询 | `queryDataList(Map 过滤)` / `(Map, 排序Map)` / `(Map, 排序, 页, 页大小)` | 主查询，返回 DataList |
| 查询 | `queryDataList(String sql)` / `queryDataListWithParams(sql, List params)` | SQL 查询 |
| 查询 | `queryWithPage(sql, keyField, 页, 页大小)` | 分页查询 |
| 查询 | `queryCount(Map 过滤)` / `countByQuerySql(sql)` | 计数 |
| 查询 | `querySummaryResult(List 汇总字段, Map 过滤)` | 聚合（如 `["{amount__sum}"]`） |
| 查询 | `fetch(字段, List 值)` / `fetchAll()` | 批量/全量取 |
| 取单条 | `getByKeyField(id)`（int/Long/String 重载） / `findByKeyField(id)` | 按主键取 DataObject |
| 写 | `insertByMap(Map)` → BehaviorResult | 按 Map 插入（业务插入，走模型逻辑） |
| 写 | `insert(String 表, Map)` → long | 直接向表插入，返回自增 id |
| 写 | `deleteByMap(Map 条件)` / `delete(String 表, Map 条件)` → int | 按条件删除 |
| 写 | `update(String 表, Map 值, Map 条件)` | 直接表更新 |
| 写 | `upsert(String 表, Map keys, Map values)` | 有则更新无则插入 |
| SQL | `executeSql(sql)` / `sqlExecute(sql)` / `getDataSource()` | 执行 SQL / 拿数据源 |
| 日志/备注 | `addLog(label, id, userId, Map 参数, invoker)` | 写操作日志 |
| 日志/备注 | `addRemark(关联id, 备注)` → long / `queryRemarkList(id, processId)` / `queryLog(keyValue, 页)` | 备注/日志查询 |
| 事件 | `sendModelEvent(msg)` / `sendDelayedModelEvent(msg, 延迟秒)` | 发模型事件 |
| 调用 | `invokeFunc(方法名, 参数...)` / `invokeApi/invokeApiAsMap/invokeApiAsList(api名, Map)` | 调模型方法/接口 |
| 元信息 | `getAction(名)` / `getFieldDef` / `getMappingDef` / `getJointDef` / `getReferDef` / `getTable()` / `getKeyField()` / `getModelDescription()` / `getDataModelOfSameSource(模型名)` | 元数据 |

#### DataObject（单条行）

| 类别 | 方法 | 说明 |
|------|------|------|
| 取值 | `getValue(字段)` / `getValueOrDefault(字段, 默认)` | 取原始值（推荐，字段无则 null） |
| 取值 | `get(字段)` → PropertyValue（`.value` / `.display`） | 取包装对象，`.value` 值、`.display` 映射显示 |
| 取值 | `getString(字段[,默认])` / `getInt(字段[,默认])` / `getLong(字段[,默认])` / `getBigDecimal(字段)` / `getDate(字段)` / `getBoolean(字段)` | 类型化取值 |
| 取值 | `getDisplay(字段)` / `getDisplayOrDefault(字段, 默认)` | 取显示值 |
| 主键 | `getKeyValue()` → Long / `getKeyFieldValue()` → String / `getKeyObject()` / `getKeyDescription()` | 主键 |
| 写 | `update(Map)` / `updateWithoutVersion(Map)` / `delete()` | 更新/删除当前行 |
| 关联 | `referTo(关联名)` → DataObject / `referObject(名)` / `referredBy(模型, 关联[, Map 过滤])` | 引用关系导航 |
| 关联 | `getRelationshipDataList(模型[, 谓词])` / `addRelationship` / `removeRelationship` | 关系数据 |
| 备注/日志 | `addRemark(备注)` → long / `addLog(label, userId, 参数, invoker)` / `queryRemarkList()` / `queryRemarkListWithPage(页, 页大小)` | 备注/日志 |
| 流程 | `startProcess(...)` / `getRunningProcess(流程名)` / `getWorkflow()` | 工作流 |
| 其他 | `getModel()` / `sqlQuery(sql)` / `asPropertyMap()` / `getVersion()` / `getFileDisplay/getImageDisplay/getFileUrls` | 杂项 |

> ⚠️ 源码中**没有** `toMap()` / `set(k,v)` / `getDouble()` / `getTimestamp()`：转 Map 用 `asPropertyMap()`，写值走 `update([...])`。

#### DataList（查询结果列表）

| 类别 | 方法 | 说明 |
|------|------|------|
| 取元素 | `fetchOne()`（空 → null，推荐） / `one()`（取单条） / `get(下标)` / `singleValue()` | 取行 |
| 转换 | `asList()` → List / `asArray()` → 数组 / `asStream()` → Stream / `getIdList()` | 转集合 |
| 统计 | `count()` / `isEmpty()` / `calculate(字段, 运算符[, 异常msg])` | 计数/聚合 |
| 关联 | `referTo(关联名)` / `referObject(名)` / `referredBy(模型, 关联[, Map])` | 为列表批量加载关联 |
| 其他 | `getModel()` / `getList()` → DataObject[] / `copy()` | 杂项 |

> ⚠️ DataList 本身**没有** `collect/each/size/list/stream` 方法：遍历/映射请对 `asList()` 结果用 Groovy `.each{}` / `.collect{}`，流式用 `asStream()`。

**综合示例**：

```groovy
def dataModel = ctx.dataModel
def host = ctx.host

// 查询列表 → 取单条
def list = dataModel.queryDataList([status: 1, is_del: 0])   // DataList
def obj  = list.fetchOne()                                    // DataObject（空 → null）
def total = list.count()
def ids  = list.asList().collect { it.getKeyValue() }         // 对 asList() 用 Groovy collect

// 取值
def name    = obj?.getValue("name")           // 原始值
def status  = obj?.getInt("status")           // 类型化
def display = obj?.get("status")?.display     // 映射显示值

// 写
def newId = dataModel.insertByMap([name: "xxx", status: 1]).id
obj?.update([name: "new_name", status: 2])
obj?.delete()

// 关联 / joint
def refName  = obj?.referTo("owner")?.getValue("name")           // reference 导航
def jointVal = obj?.get("id#member_role_joint.role_names")?.value // joint（# 语法）
```

---

### 3.6 钩子类型全谱

Groovy 钩子都由 SubProject 按方法**参数类型**分发。除常见的 Validator/Behavior/Updator/onChange 外，还有：

| 钩子 | JSON 配置处 | 方法参数类型 | 用途 |
|------|------------|------------|------|
| Validator | `action.validator` | `ActionValidatorContext` | 执行前校验 |
| Behavior | `action.behavior`（自定义名） | `ActionBehaviorContext` | 核心业务逻辑 |
| Updator | `input.updator` | `ParameterUpdateMasterContext`（详情行用 `...DetailContext`） | 字段联动 |
| onChange | `input.on_change` | `ParameterChangeContext` | 字段变更回调 |
| page_values_func | `action.page_values_func` | `ActionPageValuesFuncContext` | property2 时计算页面初值（可读 `masterParams`） |
| customInitFunc | `action.customInitFunc` | `ActionCustomInitContext` | 动态生成/改写 ActionDef |
| 筛选联动 updator | list 筛选项的 updator | `FilterUpdateContext` | 列表筛选项级联（读 `filterValues`） |
| mapping 自定义 | `mapping_defs[].custom_func` | `MappingCustomFuncContext` | 动态生成映射值 |
| joint 自定义 | `joint_defs[].custom_function` | `JointCustomFuncContext` | 用函数替代 joint SQL |
| 批量 Validator/Behavior | 批量 action | 批量上下文 | 批量操作校验/执行 |
| 模型事件 handler | `eventSubs[].handler` | `ModelEventContext` | 数据变更事件 |
| DataInterface | 数据接口 | — | 对外数据接口 |
| ModelService | `/model/{模型}/service/{func}` | `ModelServiceContext` | 绑定模型的自定义 API（见 create-service） |

### 3.7 env 环境对象

条件表达式和 server 参数里用 `env.*`，Groovy 里可拿 `EnvDataObject`：

```groovy
def env = ctx.env                    // 或 EnvDataObject.getUserEnv()
def userId = env.getUserId()         // 当前用户 ID
def xid = env.getXid()               // 当前组织 ID
```

内置环境变量（JSON 表达式/server 参数常用）：`user_id`、`xid`、`org_id`、`date`、`month`、`now`。例：
```json
{"property": "created_by", "result": "env.user_id.value"}
{"property": "created_time", "result": "env.now.value"}
```

### 3.8 Action 执行生命周期与 Variables

一次 Action 执行的引擎顺序（理解 when/enable 与 Variables 的时机）：

```
条件检查: enabled()（是否可用）/ whenPassed()（是否显示）/ authPassed()（权限）
  → 准备: getVariables() / getPageValues()（page_values_func）
  → 校验: doValidate()   → Validator
  → 执行: doBehavior()   → Behavior
  → 后处理: fireEvent() / executeCalls()
```

**Action Variables**：action 级状态变量，property2 阶段算初值、经 onChange 更新，execute 时钩子里 `ctx.variables` 可读。JSON 声明：

```json
{
  "name": "create_order",
  "variables": [
    {"name": "discount", "defaultValue": 1.0},
    {"name": "total_amount", "defaultValue": 0}
  ]
}
```

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
        def modelName = msg.getModelName()

        // 取 Host：ctx.getHost() 与 Host.getInstance() 均可用
        def host = ctx.getHost()          // 或：def host = Host.getInstance()

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

**API 端点**（领域服务，标准/匿名两种最常用；完整 7 种品类 + 模型服务见 `create-service`）：
```
POST /general/project/{subproject}/service/{ServiceClass}/{method}            # 标准，需认证
POST /general/project/{subproject}/service/anonymous/{ServiceClass}/{method}  # 匿名，免登录
```
> 另有：内部服务 `internal_service/`、SOA `/soa/{project}/{name}/{func}`、SSE 流式 `stream/`、MVC `ctrl/`；以及**模型服务** `/general/model/{模型}/service/{func}`（`ModelServiceContext`）。

**GatewayContext 字段**：
- `ctx.getBody().get("key")` / `ctx.body.key` — body 参数（两种皆可）
- `ctx.getStringParam("key")` / `ctx.getIntParam("key", 默认)` / `ctx.getLongParam("key")` — URL 参数（类型化）
- `ctx.parameters["key"]?.getAt(0)` — URL 参数（原始，值是数组）
- `ctx.getRequest()` / `ctx.request` — HttpServletRequest

---

## 6. 常用模式

### 6.1 软删除

JSON：
```json
{
  "name": "delete",
  "when": "object != null && object.getValue('is_del') == 0",
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

### 6.5 在 Groovy 中获取过滤条件（datafilter / 列表筛选值）

> ⚠️ 「读当前列表的过滤值」**不在 `ActionBehaviorContext`**，取值方式取决于所在的 Context 类型：

**A. 页面值计算函数 `page_values_func` —— `ActionPageValuesFuncContext`**（`uniplat.host`）
用 `ctx.masterParams.get("字段")` 读列表的预过滤参数（masterParams）：

```groovy
import com.qinqinxiaobao.report.uniplat.host.ActionPageValuesFuncContext

def emplyeeInAgentNumber_func(ActionPageValuesFuncContext ctx) {
    def emplyeeInId = ctx.masterParams.get("EmplyeeInId")     // 读预过滤参数
    return ctx.dataModel.getDataSource().queryForList(
        "select ... where ea.EmplyeeInId=?", emplyeeInId)
}
```

**B. 列表筛选联动 updator —— `FilterUpdateContext`**（`uniplat.host`）
用 `ctx.getFilterValues().get("字段")` 或 `ctx.filterValues.getAt("字段")` 读当前筛选值：

```groovy
import com.qinqinxiaobao.report.uniplat.host.FilterUpdateContext

def group_updator(FilterUpdateContext ctx) {
    def sender = ctx.sender
    def groupType = ctx.getFilterValues().get("GroupType")    // 读当前筛选值
    def list = ctx.dataModel.getDataSource().queryForList("... where type=?", groupType)
    return [ext_properties: [mapping_values: list]]
}
```

**环境信息**（各 Action Context 通用）：
```groovy
def xid    = ctx.env.xid.value        // 当前组织
def userId = ctx.env.user_id.value    // 当前用户
```

---

## 7. 开发规范

1. **不创建 test 目录**：这是废弃机制（增删改查也无需 test）
2. **默认用平台内置 behavior**：insert/update/delete 不需要 Groovy 方法
3. **`when`（显示）与 `enable`（启用）分清**：引用 `object` 时**先 `object != null &&`** 再判业务条件
4. **dataFilters 用于自动过滤**：特别是 `is_del = 0`
5. **Groovy 类名与文件名一致**
6. **package 可选**（建议按分组目录声明，不声明也能跑）
7. **所有业务方法都是实例方法**，用 `def` 声明
8. **Validator 用 AssertUtils 抛异常**（或 `throw new ValidateException`）
9. **Behavior 返回 `BehaviorResult`（推荐）或字典 `[result: 0, id:, msg:]`**
10. **取值优先 `obj.getValue("字段")`**；`obj.字段.value` 直取，可空字段加 `?.`

---

## 8. 关键文件位置

| 内容 | 位置 |
|------|------|
| JSON Schema | `uniplat-main/config/schemas/datamodel.json` |
| 引擎类源码（方法权威来源） | `uniplat-main/src/main/java/.../models/meta/DataModel.java`、`.../engine/DO/DataObject.java`、`.../engine/DataList.java` |
| Context 类源码 | `uniplat-main/src/main/java/.../executor/`（Action*/Parameter* 上下文）、`.../host/`（PageValuesFunc/FilterUpdate 上下文） |
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
- [ ] 确认 `when`/`enable` 引用 object 时已 `object != null &&` 守卫
- [ ] 确认 list 的 dataFilters 软删除过滤
