# 本机运行记录与启动说明

验证日期：2026-10-24。本记录对应 `D:/20476/Pictures/JAVA/rail-ticket` 的本机学习环境。

## 已验证的结果

- 使用项目专用的 Amazon Corretto Java 17 完成聚合服务、网关及其依赖共 21 个模块的打包，构建结果为 `BUILD SUCCESS`。
- 本次构建跳过了自动化测试，不能表述成“全部单元测试通过”。
- 聚合服务和网关启动，并完成 Nacos 服务注册。
- 实际请求验证了两个服务的健康接口、通过网关查询 11 个车站、示例账号登录、Redis 登录态读取，以及退出登录后的登录态删除。
- RocketMQ NameServer、Broker 启动，应用中的五个消息消费者和生产者完成初始化。尚未验证真实支付、退款与完整购票/超时关单流程。

上述业务来自上游项目；本次工作完成了本机环境准备、启动配置及基础接口验证。

## 当前地址

| 服务 | 本机地址 / 端口 | 说明 |
| --- | --- | --- |
| 业务网关 | `http://127.0.0.1:19000` | 本机 9000 端口已有其他程序，故使用 19000 |
| 聚合服务 | `http://127.0.0.1:9005` | 用户、购票、订单、支付业务的聚合入口 |
| MySQL | `127.0.0.1:13306` | Docker 内部仍使用 3306；业务数据库为 `12306` |
| Redis | `127.0.0.1:6379` | 登录态及业务缓存 |
| Nacos | `http://127.0.0.1:8848/nacos` | 使用独立的 `rail_ticket_nacos` MySQL 数据库 |
| RocketMQ NameServer | `127.0.0.1:9876` | Broker 对外地址为 `127.0.0.1:10911` |

可以在浏览器打开 `http://127.0.0.1:19000/api/ticket-service/station/all` 查看查询结果。网关根路径没有业务首页，访问根路径返回 404 不等于服务未启动。

学习数据库中的原始示例账号是 `admin`，密码是 `admin123456`。它是项目附带的演示账号，只用于本机练习。

## 本机文件放在哪里

- Java 17：工作区 `.tools/jdk17.0.21_0`，没有替换系统默认 Java。
- Maven 依赖缓存：工作区 `.tools/maven-repository`。
- 本机运行配置与脚本：开发副本 `.local-run/`。
- 启动日志：`.local-run/aggregation.log`、`.local-run/gateway.log`，对应错误输出在 `*-error.log`。
- 本机接口验证脚本：`.local-run/smoke-test.py`；结果保存在 `.local-run/smoke-results.json`，不记录 Token。

`.local-run/` 已通过本仓库的 `.git/info/exclude` 排除，属于当前机器的运行资料，未上传到 GitHub。重新克隆仓库不会自动获得这些本机文件。

## 在这台电脑上重新启动

先启动 Docker Desktop，然后在 PowerShell 执行：

```powershell
cd D:\20476\Pictures\JAVA\rail-ticket
& .\.local-run\start-local.ps1
```

脚本启动本项目的中间件容器及两个后台 Java 服务。看到进程创建提示后，还需要等待 Spring Boot 初始化完成，再查看日志或健康接口。

代码修改后需要重新打包时，先停止正在运行的本项目服务，再加 `-Build` 启动：

```powershell
& .\.local-run\stop-local.ps1
& .\.local-run\start-local.ps1 -Build
```

在运行成功后重复验证基础接口：

```powershell
python .\.local-run\smoke-test.py
```

停止本项目：

```powershell
& .\.local-run\stop-local.ps1
```

停止脚本保留数据库和 Redis 数据卷。不要删除数据卷来“重启服务”；新建学习环境还需要初始化数据库脚本和 Nacos schema。

## 本次解决的环境问题

1. 系统默认 Java 8 与项目 Java 17 不匹配：准备独立的 Java 17，由启动脚本显式使用。
2. Windows 拒绝绑定 MySQL 的 3306 端口：使用已验证可绑定的 13306，并在本地 ShardingSphere 配置中对应修改。
3. 9000 端口被其他程序占用：网关使用 19000。
4. Nacos 旧版 Java 的容器指标兼容问题：在容器中设置对应 JVM 选项。
5. Nacos 内置 Derby 初始化超时：使用其官方镜像附带的 MySQL schema 初始化独立数据库。
6. 本地 Nacos 关闭鉴权，而原应用仍带默认用户名密码登录：本地配置清空相应凭据后，重新启动并确认服务注册成功。

这些调整都放在本机运行配置中，原项目业务代码的分层保持原样。前端运行状态另行验证，不以本页的后端接口结果代替前端验收。
