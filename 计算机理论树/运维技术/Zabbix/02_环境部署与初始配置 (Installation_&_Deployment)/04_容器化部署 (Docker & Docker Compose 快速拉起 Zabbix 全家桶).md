## 2.4 容器化部署

容器化部署基于 Docker 与 Docker Compose 实现，具备环境隔离、快速部署、跨平台兼容、编排扩展便捷的优势，适用于测试环境快速搭建、云原生环境标准化部署、多实例集群编排场景。

### 2.4.1 容器化部署核心原则

1. 必须使用 Zabbix 官方 Docker 镜像，禁止使用第三方非官方镜像，避免安全漏洞与恶意代码风险；
2. 必须配置数据持久化卷，将数据库数据、Zabbix 配置文件、日志文件挂载到宿主机持久化目录，避免容器销毁导致数据丢失；
3. 同一编排内所有组件镜像版本必须完全一致，禁止跨版本混用；
4. 生产环境必须配置容器重启策略、健康检查、资源限制，保证服务可用性。

### 2.4.2 Docker Compose 全栈编排部署

Docker Compose 可一键拉起 Zabbix 全栈组件（数据库、Zabbix Server、Web 前端、Agent 2），是容器化部署的标准方式，核心编排配置覆盖以下服务：

1. **数据库服务**：使用官方 MySQL/PostgreSQL 镜像，配置环境变量初始化数据库、root 密码、Zabbix 专用用户与密码，挂载数据卷实现持久化；
2. **Zabbix Server 服务**：使用官方`zabbix-server-mysql`/`zabbix-server-pgsql`镜像，配置数据库连接环境变量、监听端口、时区配置，依赖数据库服务启动，挂载配置文件与日志目录；
3. **Zabbix Web 前端服务**：使用官方`zabbix-web-nginx-mysql`/`zabbix-web-apache-mysql`镜像，配置 Server 地址、数据库连接、时区、PHP 参数，映射 80/443 端口，依赖 Server 与数据库服务；
4. **Zabbix Agent 2 服务**：使用官方`zabbix-agent2`镜像，配置 Server 地址、主动注册地址、主机名，挂载宿主机相关目录实现主机指标采集，依赖 Server 服务。

### 2.4.3 部署后验证与配置

1. 执行`docker compose up -d`启动全栈服务，通过`docker compose ps`验证所有服务运行状态；
2. 通过浏览器访问宿主机映射的 Web 端口，完成初始化配置，使用默认管理员账号（用户名`Admin`，密码`zabbix`）登录，立即修改默认密码；
3. 验证 Agent 2 与 Server 的通信状态，配置本地主机监控，验证数据采集正常。
