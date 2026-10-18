# 关键源码入口

这些链接指向仓库内实际存在的文件。阅读时从业务入口进入，不必一次读完所有公共组件。

## 工程与启动

- [Maven 根项目](../../pom.xml)
- [框架版本与依赖管理](../../dependencies/pom.xml)
- [聚合服务入口](../../services/aggregation-service/src/main/java/org/opengoofy/index12306/biz/aggregationservice/AggregationServiceApplication.java)
- [聚合服务配置](../../services/aggregation-service/src/main/resources/application.yaml)
- [聚合服务 MySQL 与分片配置](../../services/aggregation-service/src/main/resources/shardingsphere-config-dev.yaml)
- [网关配置](../../services/gateway-service/src/main/resources/application.yaml)
- [聚合模式路由](../../services/gateway-service/src/main/resources/application-aggregation.yaml)

## Spring MVC 与登录

- [Web 模块依赖](../../frameworks/web/pom.xml)：确认 MVC 的依赖入口。
- [登录 Controller](../../services/user-service/src/main/java/org/opengoofy/index12306/biz/userservice/controller/UserLoginController.java)
- [登录 Service 实现](../../services/user-service/src/main/java/org/opengoofy/index12306/biz/userservice/service/impl/UserLoginServiceImpl.java)
- [购票 Controller](../../services/ticket-service/src/main/java/org/opengoofy/index12306/biz/ticketservice/controller/TicketController.java)

思考：请求如何映射到方法？参数从哪里来？Service 如何获得依赖？异常如何变成接口响应？

## Redis 缓存与抢票并发

- [缓存代理及 safeGet](../../frameworks/cache/src/main/java/org/opengoofy/index12306/framework/starter/cache/StringRedisTemplateProxy.java)
- [购票业务实现](../../services/ticket-service/src/main/java/org/opengoofy/index12306/biz/ticketservice/service/impl/TicketServiceImpl.java)
- [余票令牌控制](../../services/ticket-service/src/main/java/org/opengoofy/index12306/biz/ticketservice/service/handler/ticket/tokenbucket/TicketAvailabilityTokenBucket.java)
- [取令牌 Lua](../../services/ticket-service/src/main/resources/lua/ticket_availability_token_bucket.lua)
- [回滚令牌 Lua](../../services/ticket-service/src/main/resources/lua/ticket_availability_rollback_token_bucket.lua)

思考：击穿为什么需要锁后再查？普通 get 与 safeGet 有什么区别？未命中且数据库也不存在时，调用方怎样处理？Lua 原子执行的边界在哪里？

## RocketMQ 与幂等

- [超时关闭订单消费者](../../services/ticket-service/src/main/java/org/opengoofy/index12306/biz/ticketservice/mq/consumer/DelayCloseOrderConsumer.java)
- [MQ 幂等处理](../../frameworks/idempotent/src/main/java/org/opengoofy/index12306/framework/starter/idempotent/core/spel/IdempotentSpELByMQExecuteHandler.java)

思考：幂等标记的有效期与消息重试窗口是否匹配？重复投递与业务执行到一半失败有什么区别？关闭订单、释放座位、恢复 Redis 是否具有同一个事务边界？

## MySQL

- [聚合版建表脚本](../../resources/db/12306-springboot.sql)
- [聚合版样例数据](../../resources/data/12306-springboot.sql)

先画用户、乘车人、车次/座位、订单、支付之间的关系，再研究分片规则。不要把样例数据导入已有业务数据库。
