---
name: create-service
description: 创建 Uniplat 领域服务（DomainService）或模型服务（ModelService）
version: 1.0.0
---

# 创建 Uniplat 服务

> 本 skill 用于创建自定义 API 服务，支持两种类型：
> - **DomainService**：绑定到领域，适合通用业务接口
> - **ModelService**：绑定到具体模型，自动获取 DataModel

## 参数解析

`$ARGUMENTS` 应包含：
- **服务类型**（必须）：`domain` 或 `model`
- **服务名**（必须）：大驼峰格式，如 `UserService`、`OrderService`
- **方法名**（可选）：初始方法名，如 `query`、`update_status`
- **领域路径/模型名**（可选）：取决于服务类型

示例：
- `domain User query` → 创建用户领域服务
- `model Order batch_approve` → 创建订单模型服务

## 服务类型对比

| 对比项 | DomainService | ModelService |
|--------|--------------|--------------|
| 绑定对象 | 领域（SubProject） | 具体 DataModel |
| 文件位置 | `services/ServiceName.groovy` | `models/ModelName/ModelName.groovy` |
| 上下文类 | `GatewayContext` | `ModelServiceContext` |
| API 端点 | `/general/project/{project}/service/{name}/{func}` | `/general/model/{modelName}/service/{func}` |
| 适用场景 | 跨模型操作、通用业务接口 | 针对单个模型的扩展操作 |

## 执行流程

### DomainService 创建

#### 1. 确认信息

```
服务名：User
领域目录：/path/to/domain
方法：query, update_status, delete
是否需要 Before 回调：是/否
```

#### 2. 创建服务文件

文件路径：`<领域目录>/services/<ServiceName>.groovy`

```groovy
/**
 * {{ServiceName}} 领域服务
 *
 * API 端点：
 * POST /general/project/{project}/service/{{serviceName}}/{func}
 */

import com.qinqinxiaobao.report.uniplat.gateway.GatewayContext
import com.qinqinxiaobao.report.uniplat.host.Host
import com.qinqinxiaobao.report.uniplat.engine.DO.DataObject
import com.qinqinxiaobao.report.uniplat.engine.DO.EnvDataObject

class {{ServiceName}} {

    /**
     * {{methodLabel}}
     *
     * 请求参数（body）：
     * - param1: 说明
     * - param2: 说明
     *
     * 返回：
     * - 结果说明
     */
    static Object {{methodName}}(GatewayContext ctx) {
        Host host = Host.getInstance()
        EnvDataObject env = EnvDataObject.getUserEnv()

        // 获取参数（注意：parameters 是 Map<String, String[]>）
        String param1 = ctx.parameters["param1"]?.getAt(0)
        // 获取 body 参数
        Map body = ctx.body
        String param2 = body.param2

        // 业务逻辑
        DataModel model = host.getDataModel("{{RelatedModel}}")
        List<DataObject> rows = model.findByMap([field: param1])

        // 返回结果（平台自动包装为 ApiResult）
        return rows
    }

    /**
     * 另一个方法示例
     */
    static Object {{anotherMethod}}(GatewayContext ctx) {
        Map body = ctx.body

        // 获取数据模型
        DataModel model = Host.getInstance().getDataModel("{{RelatedModel}}")

        // 查询数据
        DataObject row = model.findByKey(body.id)
        if (!row) {
            return [success: false, message: "数据不存在"]
        }

        // 更新数据
        row.put("status", body.status)
        row.put("updated_at", new Date())
        row.update()

        return [success: true, data: row]
    }
}
```

#### 3. 创建 Before 回调（可选）

文件路径：`<领域目录>/services/Before<ServiceName>.groovy`

```groovy
/**
 * {{ServiceName}} 前置回调
 *
 * 在主方法执行前运行
 * 返回 null 继续执行，返回非 null 则中断
 */

import com.qinqinxiaobao.report.uniplat.gateway.GatewayContext

class Before{{ServiceName}} {

    /**
     * {{methodName}} 前置回调
     */
    static Object {{methodName}}(GatewayContext ctx) {
        // 权限检查示例
        EnvDataObject env = EnvDataObject.getUserEnv()
        if (!env.getUserId()) {
            return [success: false, message: "未登录"]
        }

        // 参数预处理
        // ctx.body.param1 = ctx.body.param1?.trim()

        return null  // 继续执行主方法
    }
}
```

### ModelService 创建

#### 1. 确认信息

```
模型名：Order
方法：batch_approve, export
是否需要匿名访问：是/否
```

#### 2. 在模型 Groovy 中添加方法

读取 `<领域目录>/models/<ModelName>/<ModelName>.groovy`，添加 ModelService 方法：

```groovy
import com.qinqinxiaobao.report.uniplat.models.service.ModelServiceContext

class {{ModelName}} {

    // ... 现有的 Validator / Behavior / Updator ...

    /**
     * {{methodLabel}}
     *
     * API 端点：
     * POST /general/model/{{ModelName}}/service/{{methodName}}
     */
    static Object {{methodName}}(ModelServiceContext ctx) {
        DataModel model = ctx.model  // 自动绑定的 DataModel
        EnvDataObject env = ctx.env
        Map body = ctx.body

        // 业务逻辑
        List ids = body.ids
        ids.each { id ->
            DataObject row = model.findByKey(id)
            if (row) {
                row.put("status", "approved")
                row.update()
            }
        }

        return [success: true, count: ids.size()]
    }
}
```

#### 3. 匿名访问（可选）

如果需要匿名访问，方法签名改用 `AnonymousModelServiceContext`：

```groovy
import com.qinqinxiaobao.report.uniplat.models.service.AnonymousModelServiceContext

// 匿名方法
static Object public_query(AnonymousModelServiceContext ctx) {
    // 无需认证即可访问
    // API: POST /general/model/Order/service/anonymous/public_query
}
```

---

## 服务类型与端点对照表

| 服务类型 | 文件位置 | API 端点 | 上下文类 |
|---------|---------|---------|---------|
| 标准领域服务 | `services/Name.groovy` | `/general/project/{project}/service/{name}/{func}` | `GatewayContext` |
| 匿名领域服务 | 同上，方法参数用 `AnonymousGatewayContext` | `/general/project/{project}/service/anonymous/{name}/{func}` | `AnonymousGatewayContext` |
| 内部领域服务 | 同上 | `/general/project/{project}/internal_service/{name}/{func}` | `InternalDomainServiceContext` |
| 模型服务 | `models/Name/Name.groovy` | `/general/model/{modelName}/service/{func}` | `ModelServiceContext` |
| 匿名模型服务 | 同上 | `/general/model/{modelName}/service/anonymous/{func}` | `AnonymousModelServiceContext` |
| SSE 流式 | `services/Name.groovy` | `/general/project/{project}/stream/{name}/{func}` | `StreamApiContext` |

---

## 常用模式

### 分页查询

```groovy
static Object query(GatewayContext ctx) {
    Host host = Host.getInstance()
    DataModel model = host.getDataModel("Order")

    // 获取分页参数
    int page = (ctx.parameters["page"]?.getAt(0) ?: "1") as int
    int size = (ctx.parameters["size"]?.getAt(0) ?: "20") as int
    int offset = (page - 1) * size

    // 构建查询条件
    Map conditions = [:]
    if (ctx.parameters["status"]?.getAt(0)) {
        conditions.status = ctx.parameters["status"][0]
    }

    // 查询总数
    int total = model.count(conditions)

    // 查询数据
    List<DataObject> rows = model.findByMap(conditions, [offset: offset, limit: size])

    return [
        success: true,
        data: rows,
        total: total,
        page: page,
        size: size
    ]
}
```

### 文件上传处理

```groovy
static Object upload(GatewayContext ctx) {
    Map body = ctx.body

    // 文件在 body 中可能是 File 对象或文件路径
    def file = body.file

    // 使用平台文件系统存储
    Host host = Host.getInstance()
    String fileId = host.getFileSystem("default").save(file)

    return [success: true, fileId: fileId]
}
```

### 事务处理

```groovy
static Object batch_operation(GatewayContext ctx) {
    Host host = Host.getInstance()
    DbHandler db = DbHandler.getInstance()

    // 事务内操作
    db.transaction {
        // 操作1
        DataModel orderModel = host.getDataModel("Order")
        DataObject order = orderModel.findByKey(ctx.body.orderId)
        order.put("status", "processing")
        order.update()

        // 操作2
        DataModel logModel = host.getDataModel("OperationLog")
        DataObject log = logModel.newObject()
        log.put("action", "batch_operation")
        log.put("target_id", ctx.body.orderId)
        log.update()

        // 如果任何操作失败，整个事务回滚
    }

    return [success: true]
}
```

### 跨领域调用

```groovy
static Object cross_domain(GatewayContext ctx) {
    Host host = Host.getInstance()

    // 调用本领域模型
    DataModel orderModel = host.getDataModel("Order")
    DataObject order = orderModel.findByKey(ctx.body.orderId)

    // 调用其他领域模型
    DataModel userModel = host.getDataModel("User")
    DataObject user = userModel.findByKey(order.getString("user_id"))

    // 调用其他领域的服务
    SubProject sub = host.getSubProject("user_project")
    Object result = sub.invokeService("UserService", "check_permission", [userId: user.getKeyValue()])

    return result
}
```

---

## 向用户确认

生成完成后，展示：
- 服务文件路径
- 方法列表及端点
- API 调用示例

```
服务已创建：
  文件：services/User.groovy
  方法：
    - query: POST /general/project/my_domain/service/User/query
    - update_status: POST /general/project/my_domain/service/User/update_status

  调用示例：
    curl -X POST http://localhost:8000/general/project/my_domain/service/User/query \
      -H "Content-Type: application/json" \
      -d '{"name": "张三"}'
```

---

## 注意事项

1. **服务名大驼峰**：`UserService`，文件名与服务名一致
2. **方法名小写下划线**：`update_status`，不能是 `updateStatus`
3. **所有方法必须 static**：Uniplat 的服务方法都是静态方法
4. **GatewayContext.parameters 是数组**：取值用 `?.getAt(0)`
5. **返回 null 或 Map/List**：平台自动包装为 ApiResult
6. **Before 回调返回非 null 会中断**：利用这个特性做权限检查、参数校验
