## 2.3 操作系统包管理安装部署

包管理安装是官方推荐的标准化部署方式，基于 Linux 发行版原生包管理器（Yum/Dnf、Apt）实现，具备安装便捷、版本统一、依赖自动管理、升级维护简单的优势，是生产环境优先选择的部署方式。

### 2.3.1 RHEL/CentOS Stream 系列（Yum/Dnf）部署流程

1. **导入官方软件源与 GPG 密钥**：添加 Zabbix 官方 LTS 版本仓库，验证 GPG 密钥保证安装包完整性，禁用第三方非官方仓库避免版本冲突；
2. **安装核心组件包**：安装`zabbix-server-mysql`/`zabbix-server-pgsql`（Server 端）、`zabbix-web-mysql`/`zabbix-web-pgsql`（Web 前端）、`zabbix-agent2`（客户端）、`zabbix-sql-scripts`（数据库 Schema 脚本）；
3. **数据库初始化**：创建 Zabbix 专用数据库与用户，导入官方 Schema 脚本，完成数据库表结构与初始数据加载；
4. **核心配置初始化**：修改`zabbix_server.conf`配置数据库连接参数，修改 PHP 配置文件满足前端运行要求；
5. **服务启动与开机自启配置**：启动`zabbix-server`、`zabbix-agent2`、Web 服务器、`php-fpm`服务，配置开机自启，验证服务运行状态；
6. **Web 前端初始化向导**：通过浏览器访问 Zabbix Web 前端，按照向导完成环境校验、数据库连接配置、Server 地址配置、管理员账号初始化，完成安装验证。

### 2.3.2 Debian/Ubuntu 系列（Apt）部署流程

1. **更新系统依赖与导入官方源**：更新系统包索引，安装`apt-transport-https`、`gnupg`等依赖包，导入 Zabbix 官方 GPG 密钥，添加对应 LTS 版本的官方软件仓库；
2. **核心组件安装**：通过`apt install`命令安装`zabbix-server-mysql`/`zabbix-server-pgsql`、`zabbix-frontend-php`、`zabbix-apache-conf`/`zabbix-nginx-conf`、`zabbix-agent2`、`zabbix-sql-scripts`；
3. **数据库初始化与配置**：创建专用数据库与用户，导入官方 Schema 脚本，配置 Server 端数据库连接参数；
4. **Web 环境配置**：修改 Apache/Nginx 站点配置，指定 Web 前端目录与监听端口，更新 PHP 配置参数，重启 Web 服务与 PHP 解析服务；
5. **服务启动与验证**：启动并启用`zabbix-server`、`zabbix-agent2`服务，完成 Web 前端初始化向导，验证监控系统可用性。
