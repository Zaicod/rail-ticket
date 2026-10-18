# 从苍穹外卖理解 12306：分层与调用链对照

继续使用 12306 票务业务和现有代码结构，以苍穹外卖的知识基础逐步阅读。本文基于当前本地源码静态核对，接口路径是阅读入口，尚未在本机启动验证。

## 1. 先分清模块和分层

`sky-common / sky-pojo / sky-server` 是工程如何拆分；`Controller / Service / Mapper` 是业务代码如何分工。这是两个维度。

12306 把业务按用户、购票、订单、支付拆成模块，每个模块里面仍然有熟悉的业务分层。

| 苍穹外卖中的概念 | 12306 中的位置 | 阅读方式 |
| --- | --- | --- |
| `sky-common` | `frameworks` 下的多个公共组件，以及各业务模块的 `common`、`toolkit` | 相当于公共能力拆得更细；不必一开始逐个阅读 |
| `sky-pojo` | 各业务模块的 `dao/entity`、`dto/req`、`dto/resp` 等 | 数据对象跟随业务模块存放；没有集中到一个 pojo 模块 |
| `sky-server` | `services/user-service`、`ticket-service`、`order-service`、`pay-service` | 理解为按业务拆开的后端模块，每个内部继续分层 |
| 应用启动类 | `services/aggregation-service` 的启动类 | 聚合入口把上述业务装配进同一个 Spring Boot 应用 |
| Nginx 反向代理的部分作用 | `services/gateway-service` 的路由能力 | 只作职责类比；Spring Cloud Gateway 和 Nginx 不是同一种组件 |

`frameworks` 是仓库作者封装的公共组件集合，项目仍然使用 Spring Boot / Spring MVC。`dependencies/pom.xml` 主要统一依赖版本，不承担 Controller 或 Service 职责。

## 2. 在用户模块里找到熟悉的目录

打开 `services/user-service/src/main/java/org/opengoofy/index12306/biz/userservice`：

```text
userservice/
├── controller/       接收 HTTP 请求，调用业务接口
├── service/          业务接口
│   └── impl/         业务实现
├── dao/
│   ├── mapper/       数据库访问接口
│   └── entity/       数据库实体（常以 DO 结尾）
├── dto/
│   ├── req/          请求参数
│   └── resp/         返回数据
├── config/           配置
├── common/           业务常量等
└── toolkit/          工具
```

`dao` 是数据库访问相关代码的组织目录，不代表在 Service 和 Mapper 之间又增加了一个必须调用的类。

| 熟悉的名字 | 当前项目的具体例子 | 对应关系 |
| --- | --- | --- |
| Entity | `UserDO` | 这是映射数据库用户表的数据对象；DO 表示 Data Object |
| 登录 DTO | `UserLoginReqDTO` | 接收用户名/邮箱/手机号与密码 |
| 登录 VO | `UserLoginRespDTO` | 在该接口承担返回对象的职责；不是所有 RespDTO 都必须改名为 VO |
| Mapper | `UserMapper` | 负责数据库访问 |
| Result | `frameworks/convention/.../result/Result.java` | 统一响应结构，包含 code、message、data、requestId；字段和成功码需要按本项目理解 |
| 工具类 / 配置类 | `toolkit`、`config` 及公共组件 | 阅读时从调用点进入 |

## 3. 第一条链路：登录

入口：`POST /api/user-service/v1/login`。

```text
请求 JSON
  → UserLoginReqDTO
  → UserLoginController.login()
  → UserLoginService.login() 接口
  → UserLoginServiceImpl.login() 实现
  → UserMailMapper / UserPhoneMapper 解析账号（按实际分支）
  → UserMapper.selectOne() 查询用户
  → JWTUtil 生成 Token
  → DistributedCache.put() 保存登录态
  → UserLoginRespDTO
  → Results.success() 包成 Result 后返回
```

实际 Controller 核心代码只有：

```java
@PostMapping("/api/user-service/v1/login")
public Result<UserLoginRespDTO> login(@RequestBody UserLoginReqDTO requestParam) {
    return Results.success(userLoginService.login(requestParam));
}
```

这与苍穹外卖的 Controller 调用 Service 是同样的分工。`@RequiredArgsConstructor` 配合 `private final UserLoginService` 生成构造方法，Spring 通过构造方法注入依赖；不能因为没看到字段上的 `@Autowired` 就认为没有注入。

阅读顺序：

1. [UserLoginReqDTO](../../services/user-service/src/main/java/org/opengoofy/index12306/biz/userservice/dto/req/UserLoginReqDTO.java)：前端传什么。
2. [UserLoginController](../../services/user-service/src/main/java/org/opengoofy/index12306/biz/userservice/controller/UserLoginController.java)：路径、参数接收、返回包装。
3. [UserLoginService](../../services/user-service/src/main/java/org/opengoofy/index12306/biz/userservice/service/UserLoginService.java) 与 [实现](../../services/user-service/src/main/java/org/opengoofy/index12306/biz/userservice/service/impl/UserLoginServiceImpl.java)：业务接口如何交给实现类执行。
4. [UserMapper](../../services/user-service/src/main/java/org/opengoofy/index12306/biz/userservice/dao/mapper/UserMapper.java) 与 [UserDO](../../services/user-service/src/main/java/org/opengoofy/index12306/biz/userservice/dao/entity/UserDO.java)：查哪个实体。
5. [UserLoginRespDTO](../../services/user-service/src/main/java/org/opengoofy/index12306/biz/userservice/dto/resp/UserLoginRespDTO.java)：前端拿到什么。

### 为什么在 XML 里找不到登录 SQL？

这里使用 MyBatis-Plus。`UserMapper extends BaseMapper<UserDO>`，所以 `selectOne()` 等通用方法由框架提供。查询条件由 Service 中的 `LambdaQueryWrapper` 组织，不要求 XML 里有同名 `<select>`。

项目仍有 [UserMapper.xml](../../services/user-service/src/main/resources/mapper/UserMapper.xml)，用于自定义操作。不能把“找不到登录的 XML SQL”理解为没有查询数据库。

## 4. 第二条链路：查票，再读购票

从 [TicketController](../../services/ticket-service/src/main/java/org/opengoofy/index12306/biz/ticketservice/controller/TicketController.java) 进入 [TicketServiceImpl](../../services/ticket-service/src/main/java/org/opengoofy/index12306/biz/ticketservice/service/impl/TicketServiceImpl.java)：

| 接口 | 当前 Controller 调用的方法 | 第一遍理解什么 |
| --- | --- | --- |
| `GET /api/ticket-service/ticket/query` | `pageListTicketQueryV1()` | 校验参数 → 查缓存 → 必要时查询数据库 → 组装返回数据 |
| `POST /api/ticket-service/ticket/purchase` | `purchaseTicketsV1()` | 业务校验 → 按列车加锁 → 执行购票 |
| `POST /api/ticket-service/ticket/purchase/v2` | `purchaseTicketsV2()` | 在同一个业务上研究令牌控制、锁粒度与优化 |

查询接口当前调用 V1，不会因为 Service 中存在 V2 就自动切换。先读入口实际调用的版本，再作比较。

从 `purchaseTicketsV1()` 继续进入 `executePurchaseTickets()`，可以看到读取列车信息、分配座位、保存车票记录，以及通过 `ticketOrderRemoteService.createTicketOrder()` 创建订单。

跨业务的 `remote` 接口是 Feign 客户端，不是访问本地表的 Mapper。其职责是调用另一组 HTTP 业务接口。理解这一点，就能区分：

```text
Service → Mapper：访问数据库
Service → Remote 接口：调用另一项业务的 API
```

这里的 `ticketService` 字段在启动回调中通过 `ApplicationContextHolder.getBean(TicketService.class)` 获取容器中的 Bean，目标方法有事务注解。不要在初读时随手改成 `this.executePurchaseTickets()`，后续学习事务代理时再研究这一点。

## 5. 聚合服务为什么能用其他模块的代码？

[聚合服务 POM](../../services/aggregation-service/pom.xml) 引用了用户、购票、订单、支付四个模块；[启动类](../../services/aggregation-service/src/main/java/org/opengoofy/index12306/biz/aggregationservice/AggregationServiceApplication.java) 配置了这些模块的组件扫描、Mapper 扫描和 Feign 扫描。

所以聚合服务不是把代码复制进去，而是通过 Maven 依赖和 Spring 扫描将业务装配到一个进程。它仍保留了部分 HTTP 调用边界：[订单 Remote 接口](../../services/ticket-service/src/main/java/org/opengoofy/index12306/biz/ticketservice/remote/TicketOrderRemoteService.java) 的目标地址可由 `aggregation.remote-url` 指定；当前聚合配置指向自身地址。同进程不等于每一次业务协作都变成 Java 方法直调，也不等于自动共享一个数据库事务。

## 6. 公共组件按用到的顺序学

1. Controller 返回 `Results.success()`：再去看 `frameworks/web` 和 `frameworks/convention`。
2. 登录保存 Redis 数据：再去看 `DistributedCache` 及其实现。
3. 看到 `safeGet()`：再研究互斥重建与双重检查，这是进入缓存击穿的入口。
4. 查票/购票碰到责任链：先了解每个校验负责什么，再看 `frameworks/designpattern` 如何组织它们。
5. 看到 `@Idempotent`：再研究重复请求/消息去重与异常处理。
6. 最后扩展到分库分表、令牌控制、消息补偿和性能验证。

第一遍阅读时，把公共组件当作已有工具接口，先弄清输入、输出和调用目的，再深入内部实现。

## 7. 这一阶段的完成标准

- 能找到登录接口的 Controller、Service、实现类、Mapper、DO、请求 DTO 和响应 DTO。
- 能解释模块拆分与业务分层的区别。
- 能解释为什么 `selectOne()` 不一定对应手写 XML。
- 能根据 TicketController 判断正在执行 V1 还是 V2。
- 能区分普通 Service 方法、Mapper 方法与 Feign Remote 方法。

这些是源码阅读目标，不等于业务已在本机跑通。完成后继续 [技术栈学习路线](roadmap.md)。
