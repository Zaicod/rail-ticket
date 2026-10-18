# RailTicket 起步指南

## 当前状态

- 上游：`nageoffer/12306`。
- 个人仓库：`Zaicod/rail-ticket`，为公开 Fork。
- 导入基线：`daf3a2714c00ff263764d23ab10c2eaed2ae842d`。
- 已完成：原版参考副本、开发副本、Git 远程关联、学习资料整理。
- 尚未完成：本机编译、依赖服务启动、业务闭环验证、Docker 编排、性能测试。

## 先用哪个版本

先使用 `services/aggregation-service` 聚合版本，配合 `services/gateway-service`。聚合版本把用户、购票、订单、支付装配到一个应用中，但仍有 Nacos、Redis、MySQL、RocketMQ 等依赖，不是无中间件的单体。

暂时把精力放在登录、查询、购票、订单取消四条链路。支付沙箱、分库分表、动态线程池和监控按需逐步理解。

## 本机环境检查记录

2026-10-18 检查到：

| 项目 | 状态 |
| --- | --- |
| Git | 已安装，GitHub 身份可用 |
| JDK | 用户目录中有 JDK 8 和 25，未找到 JDK 17；命令行未找到 `java` |
| Maven | 命令行未找到独立 Maven；仓库自带 Maven Wrapper 3.9.2 |
| Docker | 命令行未找到，尚未确认安装与运行状态 |
| MySQL / Redis / RocketMQ / Nacos | 尚未验证本机或虚拟机服务是否可用 |

项目当前依赖配置为 Java 17、Spring Boot 3.0.7、Spring Cloud 2022.0.3、Spring Cloud Alibaba 2022.0.0.0-RC2。先用 JDK 17 建立基线，避免把 JDK 25 的兼容性问题混入学习过程。不要在首次导入时同时升级所有框架。

## 环境准备后的启动顺序

以下是待执行步骤，不代表已在本机运行成功。

1. 为 IDEA 的 Project SDK 和 Maven Runner 选择 JDK 17，打开仓库根目录的 `pom.xml`。
2. 检查聚合服务的 `application.yaml`、`shardingsphere-config-dev.yaml` 以及网关配置，明确每个中间件的地址和凭据。
3. 准备 MySQL、Redis、RocketMQ 和 Nacos；再根据动态线程池、Sentinel 等实际配置补充依赖配置。现有示例凭据只能作为本地示例使用。
4. 先阅读 `resources/db/12306-springboot.sql` 建表脚本，再检查 `resources/data/12306-springboot.sql` 数据脚本的目标数据库。只向专用学习数据库导入，不覆盖已有数据库。
5. 用 Wrapper 编译聚合服务及其依赖；单独编译网关。首次构建会下载 Maven 和依赖。

```powershell
cd D:\20476\Pictures\JAVA\rail-ticket
.\mvnw.cmd -pl services/aggregation-service -am -DskipTests package
.\mvnw.cmd -pl services/gateway-service -am -DskipTests package
```

6. 在 IDEA 启动 `AggregationServiceApplication`，默认端口为 `9005`；再启动 `GatewayServiceApplication`，默认端口为 `9000`，网关默认使用 `aggregation` 配置。
7. 先验证一个查询接口，再验证登录与下单。最后按需要连接 `console-vue` 前端。

根 POM 的构建插件会执行代码格式化，且测试默认跳过。构建后先查看 `git diff`，不要把“打包成功”当成“业务测试通过”。

## 日常开发

在 `rail-ticket` 下新建功能分支，完成一个小目标后提交。`origin` 指向个人仓库，`upstream` 指向原项目。保留 `12306-upstream` 原样用于比较。

从 [学习路线](roadmap.md) 开始，每完成一个阶段保存自己的讲解、截图或测试结果。个人功能规划见 [贡献清单](contributions.md)。
