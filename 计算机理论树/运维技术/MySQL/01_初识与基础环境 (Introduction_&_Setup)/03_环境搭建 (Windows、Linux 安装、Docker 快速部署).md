### 1.3.1 Windows 系统安装

MySQL Windows 环境提供 MSI 图形化安装包与 ZIP 免安装包两种部署方式，核心流程如下：

1. **安装前准备**：从 MySQL 官方社区下载对应版本的安装包，确认系统架构（32 位 / 64 位），关闭占用 3306 端口的程序。
2. **ZIP 免安装部署（推荐学习使用）**
    
    1. 解压安装包至目标目录，新建`my.ini`配置文件，配置`basedir`（安装目录）、`datadir`（数据目录）、`port`（端口）、`character-set-server`（字符集）核心参数。
    2. 配置系统环境变量，将安装包的`bin`目录添加至`PATH`，实现全局调用 MySQL 命令。
    3. 以管理员身份运行 CMD，执行`mysqld --initialize --console`初始化数据目录，记录控制台输出的 root 用户临时密码。
    4. 执行`mysqld --install`安装 MySQL 系统服务，执行`net start mysql`启动服务。
    5. 执行`mysql -u root -p`登录数据库，输入临时密码后，执行`ALTER USER 'root'@'localhost' IDENTIFIED BY '新密码';`修改初始密码，完成安装。
    
3. **MSI 图形化安装**：通过安装向导完成环境配置、服务安装、初始化与密码设置，全程可视化操作，适合新手入门。

### 1.3.2 Linux 系统安装

Linux 是 MySQL 生产环境的主流部署系统，主流发行版分为 RPM 系（CentOS、RHEL、Fedora）与 DEB 系（Debian、Ubuntu），核心部署流程如下：

1. **安装前准备**：卸载系统自带的 mariadb 相关包，关闭防火墙或放行 3306 端口，配置系统时间与时区。
2. **官方仓库安装（推荐）**
    
    1. 配置 MySQL 官方 yum/apt 仓库，导入官方 GPG 密钥。
    2. 执行安装命令：RPM 系执行`yum install -y mysql-community-server`，DEB 系执行`apt install -y mysql-server`。
    3. 执行`systemctl start mysqld`启动服务，执行`systemctl enable mysqld`设置开机自启。
    4. 执行`grep 'temporary password' /var/log/mysqld.log`获取 root 临时密码，执行`mysql_secure_installation`完成安全初始化，修改 root 密码、禁用匿名用户、关闭远程 root 登录、删除测试库。
    
3. **核心配置**：主配置文件为`/etc/my.cnf`（RPM 系）或`/etc/mysql/my.cnf`（DEB 系），核心配置项包括数据目录、端口、字符集、InnoDB 缓冲池、连接数等。

### 1.3.3 Docker 快速部署

Docker 容器化部署可实现环境隔离、快速启停、版本一键切换，是学习与测试环境的最优选择，核心流程如下：

1. **前置条件**：完成 Docker 环境安装与服务启动。
2. **镜像拉取**：执行`docker pull mysql:8.0`或`docker pull mysql:5.7`拉取对应版本的官方镜像。
3. **容器创建与启动**
    
    ```bash
    docker run -d \
      --name mysql8 \
      -p 3306:3306 \
      -e MYSQL_ROOT_PASSWORD=你的root密码 \
      -v /本地数据目录:/var/lib/mysql \
      -v /本地配置目录/my.cnf:/etc/mysql/my.cnf \
      --restart=always \
      mysql:8.0
    ```
    
    核心参数说明：`-p`实现宿主机与容器端口映射，`-e`设置 root 初始密码，`-v`实现数据与配置文件的持久化挂载，避免容器删除后数据丢失。
4. **验证部署**：执行`docker exec -it mysql8 mysql -u root -p`，输入密码后登录数据库，执行`SELECT VERSION();`确认版本，完成部署。
