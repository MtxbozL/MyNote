## 2.2 LAMP/LNMP 基础环境准备

Zabbix 的运行依赖 **Linux 操作系统、Web 服务器、数据库、PHP 运行环境** 四大基础组件，分为 LAMP（Linux+Apache+MySQL/MariaDB+PHP）与 LNMP（Linux+Nginx+MySQL/MariaDB+PHP）两种技术栈，二者核心配置要求统一，仅 Web 服务器配置存在差异。

### 2.2.1 操作系统基础要求

#### 2.2.1.1 发行版选型

官方兼容的主流 Linux 发行版包括：RHEL、CentOS Stream、Oracle Linux、Debian、Ubuntu、SUSE Linux Enterprise Server，生产环境优先选择企业级稳定发行版，禁止使用滚动更新发行版。

#### 2.2.1.2 系统资源配置标准

| 部署规模                          | 最低配置                      | 生产环境推荐配置                                 |
| ----------------------------- | ------------------------- | ---------------------------------------- |
| 测试环境（≤100 个监控节点，NVPS≤100）     | 2 核 CPU，4GB 内存，50GB 机械硬盘  | 4 核 CPU，8GB 内存，100GB SSD                 |
| 小规模生产（100-500 个监控节点，NVPS≤500） | 4 核 CPU，8GB 内存，100GB SSD  | 8 核 CPU，16GB 内存，200GB SSD                |
| 中大规模生产（≥500 个监控节点，NVPS≥500）   | 8 核 CPU，16GB 内存，500GB SSD | 16 核 CPU，32GB + 内存，1TB + 高性能 SSD，数据库独立部署 |

#### 2.2.1.3 系统前置配置

1. 关闭 SELinux（或配置合规的安全策略），避免端口访问、文件权限拦截；
2. 配置防火墙规则，放行 Zabbix 相关端口：Server 默认 10051/TCP，Agent 默认 10050/TCP，Web 前端 80/TCP、443/TCP；
3. 配置系统时区与时间同步，生产环境必须部署 NTP 服务，保证监控节点与 Server 时间偏差≤1s，避免数据采集、告警时间异常；
4. 优化系统文件描述符限制，生产环境建议设置 nofile≥65535，避免高并发场景下文件句柄耗尽。

### 2.2.2 数据库环境配置

数据库是 Zabbix 架构的核心存储底座，官方优先支持 MySQL/MariaDB 与 PostgreSQL，生产环境严禁使用 SQLite 等轻量级数据库。

#### 2.2.2.1 版本与引擎硬性要求

1. MySQL/MariaDB：最低版本 MySQL 8.0、MariaDB 10.5，必须使用 InnoDB 存储引擎，禁止使用 MyISAM 引擎；
2. PostgreSQL：最低版本 12，推荐 14+，可兼容 TimescaleDB 时序插件，适配大规模数据存储场景；
3. 数据库字符集必须设置为`utf8mb4`，排序规则为`utf8mb4_bin`，避免中文乱码与特殊字符存储异常。

#### 2.2.2.2 核心初始化配置

1. 为 Zabbix 创建独立数据库实例与专用用户，配置最小权限原则，仅授予专用用户对 Zabbix 数据库的全权限；
2. 导入官方提供的数据库 Schema 与初始数据：MySQL/MariaDB 需导入`create.sql.gz`，PostgreSQL 需导入`create.sql.gz`与`images.sql.gz`，禁止手动修改表结构；
3. 生产环境必须配置数据库主从复制或原生高可用架构，避免数据库单点故障导致监控系统整体不可用。

### 2.2.3 Web 服务器与 PHP 环境配置

Zabbix Web 前端基于 PHP 开发，需 Web 服务器提供 HTTP 服务与 PHP 解析能力，二者核心配置要求如下：

#### 2.2.3.1 PHP 环境硬性要求

1. 版本匹配：Zabbix 6.0 LTS 要求 PHP 7.4+，7.0 LTS 要求 PHP 8.1+，必须使用官方兼容版本；
2. 必装扩展：`bcmath`、`ctype`、`curl`、`gd`、`json`、`libxml`、`mbstring`、`mysqli`/`pgsql`、`session`、`sockets`、`xml`，缺少任一扩展将导致 Web 前端无法正常安装运行；
3. php.ini 核心参数强制配置：

| 参数                  | 强制最小值                     | 生产环境推荐值 |
| ------------------- | ------------------------- | ------- |
| max_execution_time  | 300                       | 300     |
| memory_limit        | 128M                      | 256M    |
| post_max_size       | 16M                       | 32M     |
| upload_max_filesize | 2M                        | 8M      |
| max_input_time      | 300                       | 300     |
| date.timezone       | 必须配置有效时区（如 Asia/Shanghai） | 与系统时区一致 |

#### 2.2.3.2 Web 服务器配置

1. Apache：需启用`mod_php`模块，配置站点根目录指向 Zabbix Web 前端源码目录，开启`.htaccess`支持，配置 HTTPS 与访问权限控制；
2. Nginx：需配置`php-fpm`实现 PHP 解析，配置站点规则匹配 Zabbix 前端路由，禁止直接访问配置文件与敏感目录，生产环境必须配置 HTTPS 加密访问。
