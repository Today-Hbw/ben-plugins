---
name: coding-guide
description: Uniplat 低代码平台编码规范速查。在 Uniplat 项目中进行任何开发任务时参考。
version: 1.0.0
---

# Uniplat 编码规范速查

> 本 skill 是 Uniplat 低代码平台（Java + Groovy + JSON）的编码规范。任何 Uniplat 项目的编码任务都应遵循本文规范。

## 平台概览

```
JSON 配置（模型元数据、字段、Action、视图）
    +
Groovy 脚本（Validator/Behavior/Updator/DomainService）
    =
可运行的业务系统
```

**核心思想**：Java 核心代码不写业务逻辑，所有业务通过 Groovy 钩子实现。

---

## 1. 领域目录结构

每个业务域是一个独立目录（通常是一个 Git 仓库），结构如下：

```
my_domain/
├── config.json                  # 领域基础配置
├── Entrances.groovy             # 入口菜单配置
├── models/                      # 数据模型定义（核心）
│   └── Order/
│       ├── Order.json           # 模型元数据（字段、视图、Action）
│       └── Order.groovy         # 业务逻辑钩子（Validator/Behavior/Updator）
├── services/                    # 领域服务（自定义 API）
│   └── User.groovy              # 服务名首字母大写
│   └── BeforeUser.groovy        # 前置回调（可选，命名：Before + 服务名）
├── utils/                       # 通用工具类
│   └── SystemOrgUtil.groovy
├── consts/                      # 常量定义
│   └── base/common.json
└── monitors/                    # 监控/调度任务
    └── ProcessMonitors.groovy
```

**命名规则**：
- 模型名：大驼峰（`Order`、`PartyMember`）
- Groovy 文件名：与模型/服务同名（`Order.groovy`、`User.groovy`）
- JSON 文件名：与模型同名（`Order.json`）

---

## 2. 数据对象体系

| 类 | 继承 | 用途 | 关键方法 |
|----|------|------|----------|
| **DataObject** | `extends HashMap` | 数据行对象，绑定 DataModel | `update()`, `delete()`, `referTo()`, `getString()`, `startProcess()`, `addLog()`, `getKeyValue()` |
| **PureDataObject** | `extends HashMap` | 简单键值对，Action 输入参数 | `getMap()`, `getPropertyValue()`, `getValue()` |
| **EnvDataObject** | `extends PureDataObject` | 环境对象，携带用户身份和权限 | `getRightValidator()`, `getUserId()`, `getXid()` |

### 2.1 DataObject 用法

```groovy
// 查询一条数据
DataObject row = host.getDataModel("Order").findByKey(orderId)
String name = row.getString("name")
row.put("status", "approved")
row.update()

// 新建一条数据
DataObject newRow = host.getDataModel("Order").newObject()
newRow.put("name", "新订单")
newRow.put("status", "pending")
newRow.put("created_by", env.getUserId())
newRow.update()  // 插入数据库
```

### 2.2 EnvDataObject 用法

```groovy
// 获取当前用户环境
EnvDataObject env = EnvDataObject.getUserEnv()
String userId = env.getUserId()
String xid = env.getXid()  // 当前组织 ID

// 环境变量（内置）
// user_id, xid, org_id, date, month, now
```

---

## 3. Groovy 钩子方法签名

每个模型对应一个 Groovy 文件（`ModelName.groovy`），包含以下钩子方法。
方法签名必须严格匹配，平台通过**参数类型自动识别**钩子类型。

### 3.1 Validator（业务校验）

```groovy
// 签名：ActionValidatorContext → 返回 String（null 表示通过）
static String validate(ActionValidatorContext ctx) {
    DataObject row = ctx.row
    PureDataObject inputs = ctx.inputs
    EnvDataObject env = ctx.env
    ActionDef action = ctx.action

    if (!inputs.getString("name")) {
        return "名称不能为空"
    }
    return null  // 校验通过
}
```

**上下文字段**：
- `ctx.row` — 当前数据行（DataObject）
- `ctx.inputs` — 用户输入参数（PureDataObject）
- `ctx.env` — 用户环境（EnvDataObject）
- `ctx.action` — Action 定义（ActionDef）
- `ctx.variables` — Action 变量（Map）

### 3.2 Behavior（核心业务逻辑）

```groovy
// 签名：ActionBehaviorContext → 返回 BehaviorResult 或 void
static Object doBehavior(ActionBehaviorContext ctx) {
    DataObject row = ctx.row
    PureDataObject inputs = ctx.inputs
    EnvDataObject env = ctx.env

    // 操作类型：doInsert / doUpdate / doDelete
    if (ctx.isInsert) {
        // 新增逻辑
    } else if (ctx.isUpdate) {
        // 修改逻辑
    }

    return null  // 返回 null 走平台默认持久化
    // 或返回自定义 BehaviorResult
}
```

**上下文字段**：
- `ctx.row` — 当前数据行（DataObject）
- `ctx.inputs` — 用户输入参数（PureDataObject）
- `ctx.env` — 用户环境（EnvDataObject）
- `ctx.isInsert` / `ctx.isUpdate` / `ctx.isDelete` — 操作类型标识
- `ctx.variables` — Action 变量（Map）

### 3.3 Updator（字段联动）

Updator 有 4 种签名，平台根据参数类型自动识别：

```groovy
// ① 主表上下文 Updator（推荐，新代码用这个）
static void updator(ParameterUpdateMasterContext ctx) {
    ctx.getEnv()           // EnvDataObject
    ctx.getVariables()     // Map<String, Object>
    ctx.getParameters()    // 参数列表
    ctx.getSender()        // 触发变更的字段名，首次加载为空字符串
    ctx.setData("field_name", value)  // 设置字段值
    ctx.setOptions("field_name", optionList)  // 设置下拉选项
}

// ② 详情表上下文 Updator
static void updator(ParameterUpdateDetailContext ctx) {
    ctx.getDetail()        // 详情名
    ctx.getSender()        // detail.field 格式
}

// ③ 6 参数 Updator（兼容旧代码）
static void updator(List<DataObject> datalist, EnvDataObject env,
                    Map variables, PureDataObject params,
                    List details, String sender) {
    // 旧式写法，新代码推荐用 ① 或 ②
}

// ④ 8 参数 Updator（兼容旧代码）
static void updator(List<DataObject> datalist, EnvDataObject env,
                    Map variables, PureDataObject params,
                    List details, String sender,
                    DataObject masterRow, String detailName) {
    // 旧式写法
}
```

**sender 触发时机**：

| 时机 | sender 值 |
|------|----------|
| 首次加载（property2） | `""` (空字符串) |
| 主表字段变更 | 字段名（如 `"city_id"`） |
| 详情行字段变更 | `"detail名.字段名"` |

### 3.4 onChange（字段值变更回调）

```groovy
// 签名：ParameterChangeContext
static void onChange(ParameterChangeContext ctx) {
    DataObject row = ctx.row
    EnvDataObject env = ctx.env
    Map variables = ctx.variables
    String fieldName = ctx.fieldName
    Object newValue = ctx.newValue
}
```

### 3.5 pageValuesFunc（页面值计算）

```groovy
// 签名：参数为 (DataObject, EnvDataObject)
static Object pageValuesFunc(DataObject row, EnvDataObject env) {
    // 计算列表/详情页显示值
    return null
}
```

### 3.6 customInitFunc（动态 Action 定义）

```groovy
// 签名：ActionCustomInitContext
static Object customInit(ActionCustomInitContext ctx) {
    ActionDef actionDef = ctx.actionDef
    EnvDataObject env = ctx.env
    // 动态修改 Action 定义（增删字段、修改配置等）
    return actionDef
}
```

---

## 4. Host 核心容器

Host 是平台单例，所有业务逻辑通过它获取资源：

```groovy
Host host = Host.getInstance()

// 获取数据模型
DataModel model = host.getDataModel("Order")

// 获取当前用户
AuthUser user = host.getUser()

// 获取子项目（领域容器）
SubProject sub = host.getSubProject("party_project")

// 调用工具方法
host.invokeUpdator(modelName, updator, context)
host.invokePageValuesFunc(modelName, func, row)
host.sendModelEvent(...)
```

---

## 5. 领域服务（DomainService）

绕过模型，直接定义自定义 API。

**文件位置**：`services/` 目录（可按子目录分组）

```groovy
// services/User.groovy
// 方法签名：static Object funcName(GatewayContext ctx)

import com.qinqinxiaobao.report.uniplat.gateway.GatewayContext

class User {

    static Object query(GatewayContext ctx) {
        // 获取参数（注意：parameters 是 Map<String, String[]>）
        String name = ctx.parameters["name"]?.getAt(0)
        Map body = ctx.body  // 请求体

        Host host = Host.getInstance()
        DataModel model = host.getDataModel("User")

        // 查询数据
        List<DataObject> rows = model.findByMap([name: name])

        return rows  // 平台自动包装为 ApiResult
    }

    static Object update_status(GatewayContext ctx) {
        String id = ctx.body.id
        String status = ctx.body.status

        DataObject row = host.getDataModel("User").findByKey(id)
        row.put("status", status)
        row.update()

        return [success: true]
    }
}
```

**GatewayContext 字段**：
- `ctx.host` — Host 实例
- `ctx.parameters` — `Map<String, String[]>` 查询参数（**注意是数组**）
- `ctx.body` — `Map` 请求体
- `ctx.request` — `HttpServletRequest`
- `ctx.pathValue` — 路径参数

**API 端点**：
```
POST /general/project/{project}/service/{service}/{func}
```

### 5.1 服务类型速查

| 类型 | 端点 | 上下文类 | 认证要求 |
|------|------|---------|---------|
| 标准服务 | `/general/project/{project}/service/{name}/{func}` | `GatewayContext` | 需认证 |
| 匿名服务 | `/general/project/{project}/service/anonymous/{name}/{func}` | `AnonymousGatewayContext` | 无需认证 |
| 内部服务 | `/general/project/{project}/internal_service/{name}/{func}` | `InternalDomainServiceContext` | 内部 Token |
| SOA 服务 | `/soa/{project}/{name}/{func}` | `GatewayContext` | SOA 集成 |
| SSE 流式 | `/general/project/{project}/stream/{name}/{func}` | `StreamApiContext` | 需认证 |
| MVC 控制器 | `/general/project/{domain}/ctrl/{name}/{func}` | `ControllerContext` | 需认证 |

### 5.2 Before 前置回调

在 `services/` 目录下创建 `Before + 服务名` 的文件：

```groovy
// services/BeforeUser.groovy
class BeforeUser {
    // 方法名与主服务方法名一致
    static Object query(GatewayContext ctx) {
        // 在主服务方法执行前运行
        // 可做权限检查、参数预处理等
        return null  // 返回 null 继续执行主方法
        // 返回非 null 值则中断，直接返回该值
    }
}
```

---

## 6. 模型服务（ModelService）

绑定到具体模型的自定义 API，入口是模型名，自动获取 DataModel。

```groovy
// models/Order/Order.groovy 中添加
// 方法签名：static Object funcName(ModelServiceContext ctx)

import com.qinqinxiaobao.report.uniplat.models.service.ModelServiceContext

class Order {

    // ... Validator / Behavior / Updator ...

    // 模型服务方法
    static Object batch_approve(ModelServiceContext ctx) {
        DataModel model = ctx.model  // 自动绑定的 DataModel
        List ids = ctx.body.ids

        ids.each { id ->
            DataObject row = model.findByKey(id)
            row.put("status", "approved")
            row.update()
        }

        return [success: true]
    }
}
```

**API 端点**：
```
POST /general/model/{modelName}/service/{func}
```

**ModelServiceContext 字段**：
- `ctx.model` — 自动绑定的 DataModel
- `ctx.host` — Host 实例
- `ctx.parameters` — 查询参数
- `ctx.body` — 请求体
- `ctx.env` — 用户环境

---

## 7. 事件系统

### 7.1 进程内事件

```groovy
// 监听模型事件（在 Entrances.groovy 或专用监听器中注册）
host.addListener(DataModelEvent) { event ->
    String modelName = event.modelName
    String action = event.action
    DataObject row = event.row
}
```

### 7.2 跨进程事件（RabbitMQ）

```groovy
// 发送事件
host.sendModelEvent(modelName, action, row)

// 发送延迟事件
host.sendDelayedModelEvent(modelName, action, row, delaySeconds)
```

---

## 8. 数据库操作

### 8.1 DataModel 查询

```groovy
DataModel model = host.getDataModel("Order")

// 按主键查询
DataObject row = model.findByKey(id)

// 按条件查询
List<DataObject> rows = model.findByMap([status: "pending", type: "normal"])

// 查询所有
List<DataObject> all = model.findAll()
```

### 8.2 DbHandler 直接操作

```groovy
import com.qinqinxiaobao.report.uniplat.db.DbHandler

// 获取默认数据源的 DbHandler
DbHandler db = DbHandler.getInstance()

// SQL 查询
List<Map> results = db.query("SELECT * FROM t_order WHERE status = ?", ["pending"])

// SQL 执行
db.execute("UPDATE t_order SET status = ? WHERE id = ?", ["approved", orderId])

// 指定数据源
DbHandler mysqlDb = DbHandler.getInstance("mysql")
```

---

## 9. 常见模式

### 9.1 跨模型操作

```groovy
Host host = Host.getInstance()

// 操作另一个模型
DataModel userModel = host.getDataModel("User")
DataObject user = userModel.findByKey(userId)
user.put("score", user.getInt("score") + 10)
user.update()
```

### 9.2 事务处理

```groovy
// Behavior 中的事务由平台管理
// 如需手动事务：
DbHandler db = DbHandler.getInstance()
db.transaction {
    // 事务内的操作
    order.update()
    payment.update()
}
```

### 9.3 远程调用

```groovy
// 调用另一个领域的服务
Host host = Host.getInstance()
SubProject sub = host.getSubProject("hr_project")
Object result = sub.invokeService("EmployeeService", "query", ctx)
```

### 9.4 工具类调用

```groovy
// 调用本领域工具
Object result = SystemOrgUtil.getOrgName(xid)

// 调用其他领域工具（通过 Host）
Host host = Host.getInstance()
Object result = host.invokePluginMethod("hr_project", "HrUtil", "calcSalary", [params])
```

---

## 10. 常量定义

常量文件位于 `consts/` 目录，JSON 格式：

```json
// consts/base/common.json
{
  "ORDER_STATUS": {
    "PENDING": "pending",
    "APPROVED": "approved",
    "REJECTED": "rejected"
  },
  "USER_TYPE": {
    "ADMIN": 1,
    "NORMAL": 0
  }
}
```

在 Groovy 中引用：

```groovy
// 平台会自动加载常量到上下文
String status = ORDER_STATUS.PENDING
```

---

## 11. 注意事项

1. **Groovy 方法签名必须严格匹配**：平台通过参数类型自动判断钩子类型，签名错误会导致钩子不生效
2. **GatewayContext.parameters 是数组**：`ctx.parameters["name"]` 返回 `String[]`，需要 `?.getAt(0)` 取第一个值
3. **DataObject.update() 同时处理插入和更新**：主键为空时插入，有值时更新
4. **Validator 返回 null 表示通过**：返回非 null 字符串即为错误信息
5. **Updator 的 sender 首次加载为空字符串**：用 `sender == ""` 判断是否为初始化加载
6. **不要 import 平台核心类**：Host、DataModel 等在 Groovy 中默认可用，无需 import
7. **Before 回调返回非 null 会中断主方法**：利用这个特性做权限检查

---

## 12. 快速参考

### 创建新模型的完整步骤

1. 在 `models/` 下创建 `ModelName/` 目录
2. 创建 `ModelName.json`（模型元数据）
3. 创建 `ModelName.groovy`（业务钩子）
4. 在 `config.json` 中注册模型（如需要）

### 新增 Action 的完整步骤

1. 在 `ModelName.json` 的 `actions` 数组中添加 Action 定义
2. 在 `ModelName.groovy` 中添加对应的 Validator/Behavior 方法
3. 如需字段联动，添加 Updator 方法

### 新增领域服务的完整步骤

1. 在 `services/` 下创建 `ServiceName.groovy`
2. 编写静态方法，签名 `static Object funcName(GatewayContext ctx)`
3. 如需前置处理，创建 `BeforeServiceName.groovy`
