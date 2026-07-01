---
name: create-service
description: 创建 Uniplat 领域服务（DomainService）
version: 1.0.1
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

import com.qinqinxiaobao.report.uniplat.gateway.GatewayContext
import com.qinqinxiaobao.report.uniplat.host.Host
import com.qinqinxiaobao.report.uniplat.engine.DO.DataObject
import com.qinqinxiaobao.report.utils.AssertUtils
import com.qinqinxiaobao.report.utils.StringUtils

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
    POST /general/project/{subproject}/service/passport_api/queryOrders

  调用示例：
    curl -X POST http://localhost:8000/general/project/hro_spview/service/passport_api/getUsersInfo \
      -H "Content-Type: application/json" \
      -d '{"userMemberIds": "1,2,3"}'
```

---

## 常用模式

### 分页查询

```groovy
def query(GatewayContext ctx) {
    def host = Host.getInstance()
    def body = ctx.getBody()

    // 获取分页参数
    def page = (body.get("page") ?: "1") as int
    def size = (body.get("size") ?: "20") as int

    // 构建查询条件
    def conditions = [is_del: 0]
    if (body.get("status")) {
        conditions.status = body.get("status") as int
    }

    def model = host.getDataModel("system_user")
    def list = model.queryDataList(conditions)

    // 手动分页
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

    return [
        result: 0,
        msg   : "操作成功",
        count : ids.size()
    ]
}
```

### 跨模型操作

```groovy
def crossModel(GatewayContext ctx) {
    def host = Host.getInstance()
    def body = ctx.getBody()

    // 操作多个模型
    def userModel = host.getDataModel("system_user")
    def orgModel = host.getDataModel("system_org")

    def user = userModel.getByKeyField(body.get("userId") as long)
    def org = orgModel.getByKeyField(body.get("orgId") as long)

    AssertUtils.isTrue(user != null, "用户不存在")
    AssertUtils.isTrue(org != null, "组织不存在")

    // 业务逻辑...

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
        "system_user",           // 模型名
        "getUserDisplay",        // 方法名
        body.get("userId") as long  // 参数
    )

    return [result: 0, data: result]
}
```

### 直接 SQL 查询

```groovy
import com.qinqinxiaobao.report.db.DataSourceFactory
import com.qinqinxiaobao.report.db.DbConsts

def queryWithSql(GatewayContext ctx) {
    def body = ctx.getBody()
    def status = body.get("status") as int

    // 获取数据源
    def ds = DataSourceFactory.getDataSource(DbConsts.HOST)
    // 指定数据源：DataSourceFactory.getDataSource("mssql_finance")

    // 查询
    def results = ds.queryForList(
        "SELECT * FROM system_user WHERE status = ? AND is_del = 0",
        status
    )

    return results
}
```

### 事务处理

```groovy
import com.qinqinxiaobao.report.db.DataSourceFactory
import com.qinqinxiaobao.report.db.DbConsts

def transactional(GatewayContext ctx) {
    def host = Host.getInstance()
    def body = ctx.getBody()

    DataSourceFactory.transaction(DbConsts.HOST, {
        def model = host.getDataModel("system_user")
        def obj = model.getByKeyField(body.get("id") as long)

        obj.update([status: 1])
        // 更多操作...
        // 任何异常都会回滚
    } as Runnable)

    return [result: 0, msg: "操作成功"]
}
```

### 远程调用

```groovy
def remoteCall(GatewayContext ctx) {
    def body = ctx.getBody()

    // HTTP 调用
    def url = "https://api.example.com/data"
    def conn = new URL(url).openConnection() as HttpURLConnection
    conn.requestMethod = "POST"
    conn.setRequestProperty("Content-Type", "application/json")
    conn.doOutput = true
    conn.outputStream << new groovy.json.JsonBuilder(body).toString()

    def response = conn.inputStream.text
    def result = new groovy.json.JsonSlurper().parseText(response)

    return result
}
```

---

## GatewayContext 字段速查

```groovy
ctx.getBody()                    // Map 请求体
ctx.getBody().get("key")         // 获取 body 参数
ctx.getBody().get("key")?.toString()  // 安全获取字符串
ctx.parameters                   // Map<String, String[]> URL 参数
ctx.parameters["key"]?.getAt(0)  // 取第一个 URL 参数
ctx.request                      // HttpServletRequest
ctx.request.getHeader("X-xxx")   // 获取请求头
ctx.pathValue                    // 路径参数
```

---

## 注意事项

1. **package 声明**：`package services`
2. **类名与文件名一致**：`passport_api.groovy` → `class passport_api`
3. **所有方法都是实例方法**（不是 static），用 `def` 声明
4. **参数类型是 `GatewayContext`**
5. **获取 body 用 `ctx.getBody().get("key")`**，不是 `ctx.body.key`
6. **获取 URL 参数用 `ctx.parameters["key"]?.getAt(0)`**
7. **Host.getInstance() 获取单例**，也可用 `ctx.host`
8. **返回值自动包装为 ApiResult**，直接返回 Map 或 List 即可
9. **用 AssertUtils 做参数校验**，抛异常由平台统一处理
10. **服务文件命名**：描述性 + `_api` 后缀（如 `passport_api`、`client_api`）
