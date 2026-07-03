---
name: create-service
description: 创建 Uniplat 领域服务（DomainService）
version: 1.0.3
---

# 创建 Uniplat 领域服务

> 本 skill 用于创建 Uniplat 领域服务（自定义 API），文件位于子项目的 `services/` 目录下。

## 参数解析

`$ARGUMENTS` 应包含：
- **服务名**（必须）：snake_case，如 `passport_api`、`client_api`
- **子项目路径**（可选）：子项目目录路径，省略则询问用户
- **方法名**（可选）：初始方法名，如 `getUsersInfo`、`queryOrders`

示例：
- `passport_api` → 创建 passport_api 服务
- `report_api /path/to/subproject generate_monthly` → 创建带方法的报告服务

## 执行流程

### 1. 确认信息

用 AskUserQuestion 询问：

```
服务名：passport_api
子项目目录：/path/to/subproject
方法：
  - getUsersInfo: 批量获取用户信息
  - queryOrders: 查询订单
```

### 2. 创建服务文件

文件路径：`<子项目>/services/<service_name>.groovy`

```groovy
package services

import com.example.report.uniplat.gateway.GatewayContext
import com.example.report.uniplat.host.Host
import com.example.report.uniplat.engine.DO.DataObject
import com.example.report.utils.AssertUtils
import com.example.report.utils.StringUtils

class {{service_name}} {

    /**
     * {{methodLabel}}
     *
     * 请求参数（body）：
     * - param1: 说明
     *
     * 返回：
     * - 结果说明
     */
    def {{methodName}}(GatewayContext ctx) {
        def host = Host.getInstance()

        // 获取 body 参数
        def body = ctx.getBody()
        def param1 = body.get("param1")?.toString()

        // 获取查询参数（URL 参数）
        // def queryParam = ctx.parameters["key"]?.getAt(0)

        // 获取用户信息
        def user = host.getUser()
        def userId = user.id

        // 获取数据模型
        def model = host.getDataModel("{{RelatedModel}}")

        // 查询数据
        def list = model.queryDataList([status: 1, is_del: 0])

        // 返回结果（平台自动包装为 ApiResult）
        return list.list.collect { item ->
            [
                id   : item.getKeyValue(),
                name : item.name.value,
                status: item.status.display
            ]
        }
    }

    /**
     * 另一个方法示例
     */
    def {{anotherMethod}}(GatewayContext ctx) {
        def host = Host.getInstance()
        def body = ctx.getBody()
        def id = body.get("id")?.toString()

        AssertUtils.isTrue(StringUtils.notEmptyString(id), "ID不能为空")

        def model = host.getDataModel("{{RelatedModel}}")
        def obj = model.getByKeyField(id as long)

        AssertUtils.isTrue(obj != null, "数据不存在")

        // 更新数据
        obj.update([
            status     : 1,
            update_time: host.getTimestamp()
        ])

        return [
            result: 0,
            msg   : "操作成功",
            data  : [id: obj.getKeyValue(), name: obj.name.value]
        ]
    }
}
```

### 3. 向用户确认

完成后展示：
- 服务文件路径
- 方法列表
- API 端点示例

```
服务已创建：
  文件：services/passport_api.groovy
  方法：
    - getUsersInfo
    - queryOrders

  API 端点：
    POST /general/project/{subproject}/service/passport_api/getUsersInfo
```

---

## 服务端点与类型

**领域服务（DomainService，绑定子项目/领域）**——共 7 种，路径前缀 `/general/project`（SOA 除外），方法参数用对应上下文类：

| 类型 | 端点 | 上下文类 | 认证 |
|------|------|---------|------|
| 标准服务 | `/general/project/{子项目}/service/{类}/{方法}` | `GatewayContext` | 需认证 |
| 匿名服务 | `/general/project/{子项目}/service/anonymous/{类}/{方法}` | `AnonymousGatewayContext` | 免登录 |
| 内部服务 | `/general/project/{子项目}/internal_service/{类}/{方法}` | `InternalDomainServiceContext` | 内部 Token |
| SOA 服务 | `/soa/{子项目}/{类}/{方法}` | `GatewayContext` | SOA 集成 |
| SSE 流式 | `/general/project/{子项目}/stream/{类}/{方法}` | `StreamApiContext` | 需认证 |
| 匿名流式 | `/general/project/{子项目}/stream/anonymous/{类}/{方法}` | `AnonymousStreamApiContext` | 免登录 |
| MVC 控制器 | `/general/project/{子项目}/ctrl/{类}/{方法}` | `ControllerContext` | MVC 风格 |

例（匿名服务）：`POST /general/project/biz_spview/service/anonymous/tools_api/it_year_result`

**模型服务（ModelService，绑定具体模型）**——入口是模型名，自动拿到对应 DataModel：

| 类型 | 端点 | 上下文类 |
|------|------|---------|
| 模型服务 | `/general/model/{模型}/service/{方法}` | `ModelServiceContext` |
| 匿名模型服务 | `/general/model/{模型}/service/anonymous/{方法}` | `AnonymousModelServiceContext` |

```groovy
// 模型服务写在模型的 .groovy 里，参数用 ModelServiceContext
import com.example.report.uniplat.models.service.ModelServiceContext

def exportData(ModelServiceContext ctx) {
    def model = ctx.dataModel          // 自动绑定当前模型
    def list = model.queryDataList([is_del: 0])
    return list.asList().collect { [id: it.getKeyValue(), name: it.getValue("name")] }
}
```

> **DomainService 与 ModelService 的区别**：前者绑定领域（`services/` 下、参数 `GatewayContext`），后者绑定模型（写在模型 groovy 里、参数 `ModelServiceContext`）。

---

## 常用模式

### 分页查询

```groovy
def query(GatewayContext ctx) {
    def host = Host.getInstance()
    def body = ctx.getBody()

    def page = (body.get("page") ?: "1") as int
    def size = (body.get("size") ?: "20") as int

    def conditions = [is_del: 0]
    if (body.get("status")) {
        conditions.status = body.get("status") as int
    }

    def model = host.getDataModel("system_user")
    def list = model.queryDataList(conditions)

    def total = list.count()
    def items = list.list.drop((page - 1) * size).take(size)

    return [
        result: 0,
        data  : items.collect { [id: it.getKeyValue(), name: it.name.value] },
        total : total,
        page  : page,
        size  : size
    ]
}
```

### 批量操作

```groovy
def batchUpdate(GatewayContext ctx) {
    def host = Host.getInstance()
    def body = ctx.getBody()
    def ids = body.get("ids")?.toString()?.split(",")?.collect { it.trim() as long }

    AssertUtils.isTrue(ids != null && ids.size() > 0, "请选择要操作的数据")

    def model = host.getDataModel("system_user")
    def now = host.getTimestamp()
    def operator = host.getUser().id as long

    ids.each { id ->
        def obj = model.getByKeyField(id)
        if (obj) {
            obj.update([
                status     : 1,
                update_time: now,
                update_by  : operator
            ])
        }
    }

    return [result: 0, msg: "操作成功", count: ids.size()]
}
```

### 跨模型操作

```groovy
def crossModel(GatewayContext ctx) {
    def host = Host.getInstance()
    def body = ctx.getBody()

    def userModel = host.getDataModel("system_user")
    def orgModel = host.getDataModel("system_org")

    def user = userModel.getByKeyField(body.get("userId") as long)
    def org = orgModel.getByKeyField(body.get("orgId") as long)

    AssertUtils.isTrue(user != null, "用户不存在")
    AssertUtils.isTrue(org != null, "组织不存在")

    return [result: 0, msg: "操作成功"]
}
```

### 调用模型方法

```groovy
def callModelMethod(GatewayContext ctx) {
    def host = Host.getInstance()
    def body = ctx.getBody()

    // 调用模型 Groovy 中的方法
    def result = host.invokeModelFunc(
        "system_user",
        "getUserDisplay",
        body.get("userId") as long
    )

    return [result: 0, data: result]
}
```

### 直接 SQL 查询

```groovy
import com.example.report.db.DataSourceFactory
import com.example.report.db.DbConsts

def queryWithSql(GatewayContext ctx) {
    def body = ctx.getBody()
    def status = body.get("status") as int

    def ds = DataSourceFactory.getDataSource(DbConsts.HOST)
    def results = ds.queryForList(
        "SELECT * FROM system_user WHERE status = ? AND is_del = 0",
        status
    )

    return results
}
```

### 事务处理

```groovy
import com.example.report.db.DataSourceFactory
import com.example.report.db.DbConsts

def transactional(GatewayContext ctx) {
    def host = Host.getInstance()
    def body = ctx.getBody()

    DataSourceFactory.transaction(DbConsts.HOST, {
        def model = host.getDataModel("system_user")
        def obj = model.getByKeyField(body.get("id") as long)

        obj.update([status: 1])
        // 更多操作...
    } as Runnable)

    return [result: 0, msg: "操作成功"]
}
```

---

## GatewayContext 字段速查

```groovy
// body 参数（两种写法皆可）
ctx.getBody()                     // Map 请求体
ctx.getBody().get("key")          // 取 body 参数
ctx.body.key                      // 直接属性访问 body 字段（真实项目常用）
ctx.getBody(SomeDto.class)        // 直接反序列化为 DTO

// URL 参数：类型化取参（推荐）
ctx.getStringParam("key")         // 字符串
ctx.getStringParam("key", "默认") // 带默认值
ctx.getIntParam("page", 0)        // 整数 + 默认
ctx.getLongParam("id")            // 长整数
ctx.getBigDecimalParam("amount")  // BigDecimal
ctx.parameters                    // Map<String, String[]> 原始 URL 参数（值是数组）
ctx.parameters["key"]?.getAt(0)   // 取第一个原始 URL 参数

// 其他
ctx.getRequest()                  // 或 ctx.request，HttpServletRequest
ctx.request.getHeader("X-xxx")    // 获取请求头
ctx.pathValue                     // 路径参数
ctx.host                          // Host（等价 Host.getInstance()）
```

## Host 常用方法（服务里）

```groovy
def host = ctx.host                        // 或 Host.getInstance()
host.getUser()                             // 当前用户（也可 ctx.host.user）
host.getTimestamp()                        // 当前时间戳
host.getDataModel("system_user")           // 取数据模型
host.invokeModelFunc("模型", "方法", args)  // 调用模型 Groovy 方法
host.callUtilMethod("子项目", "工具类", "方法", args...)  // 调用领域工具类方法
```

---

## 注意事项

1. **package 声明**：`services/` 直放用 `package services`；放在子目录（如 `services/system/`）则用多级包 `package services.system`
2. **类名与文件名一致**：`passport_api.groovy` → `class passport_api`
3. **方法用实例方法 `def`**（参考项目实测）；⚠️ 部分文档写 `static Object f(GatewayContext)`，以参考项目的实例 `def` 为准
4. **参数类型按服务类型选**：标准 `GatewayContext`、匿名 `AnonymousGatewayContext`、流式 `StreamApiContext`、模型服务 `ModelServiceContext`（见「服务端点与类型」）
5. **取 body**：`ctx.getBody().get("key")` 或 `ctx.body.key` 皆可；URL 参数用 `ctx.getStringParam/getIntParam/getLongParam`
6. **Host.getInstance() 获取单例**，也可用 `ctx.host`
7. **返回值**：简单场景直接返回 Map/List（平台包装为 ApiResult）；复杂/泛型可显式 `new ApiResult<>(data)`
8. **用 AssertUtils 做参数校验**
9. **服务文件命名**：描述性 + `_api` 后缀
