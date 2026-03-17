## 一、Fedora 41 部署流程

### 1. 环境准备

```bash
# 查看系统版本
cat /etc/fedora-release

# 检查并卸载旧版 MySQL/MariaDB
dnf list installed | grep mysql
dnf list installed | grep mariadb
sudo dnf remove -y mysql* mariadb*

# 关闭防火墙（生产环境可开放 3306 端口）
sudo systemctl stop firewalld
sudo systemctl disable firewalld

# 关闭 SELinux（临时+永久）
sudo setenforce 0
sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```

### 2. 安装 MySQL 8.0（官方 YUM 仓库）

bash

运行

```
# 1. 下载官方 MySQL YUM 源（Fedora 41 适配 el9 版本）
wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm

# 2. 安装 YUM 源
sudo rpm -ivh mysql80-community-release-el9-1.noarch.rpm

# 3. 安装 MySQL 服务器
sudo dnf install -y mysql-community-server
```

### 3. 初始化与启动

bash

运行

```
# 1. 初始化数据库（生成临时密码）
sudo mysqld --initialize --user=mysql

# 2. 启动 MySQL 服务
sudo systemctl start mysqld
sudo systemctl enable mysqld

# 3. 查看临时密码（在 /var/log/mysqld.log 中）
sudo grep 'temporary password' /var/log/mysqld.log
```

### 4. 配置 my.cnf（面试常考配置）

编辑 `/etc/my.cnf`：

ini

```
[mysqld]
port=3306
user=mysql
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock

# 字符集（面试重点：8.0 默认 utf8mb4）
character-set-server=utf8mb4
collation-server=utf8mb4_0900_ai_ci

# 认证插件（兼容旧版客户端）
default_authentication_plugin=mysql_native_password

# 日志配置
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
```

修改后重启：`sudo systemctl restart mysqld`

### 5. 安全设置（必做）

bash

运行

```
# 执行安全脚本，按提示操作：
# 1. 输入临时密码
# 2. 修改 root 密码（需符合强度要求：大小写+数字+特殊字符）
# 3. 移除匿名用户
# 4. 禁止 root 远程登录
# 5. 移除测试数据库
# 6. 刷新权限
sudo mysql_secure_installation
```

### 6. 远程连接配置（可选）

bash

运行

```
# 1. 登录 MySQL
mysql -u root -p

# 2. 创建远程用户（替换 username 和 password）
CREATE USER 'username'@'%' IDENTIFIED BY 'password';

# 3. 授权（所有库表权限）
GRANT ALL PRIVILEGES ON *.* TO 'username'@'%' WITH GRANT OPTION;

# 4. 刷新权限
FLUSH PRIVILEGES;

# 5. 退出后，修改 my.cnf 注释 bind-address（或改为 0.0.0.0）
# 重启 MySQL 生效
```

---

## 二、Ubuntu 部署流程

### 1. 环境准备

bash

运行

```
# 查看系统版本
lsb_release -a

# 检查并卸载旧版 MySQL
dpkg -l | grep mysql
sudo apt remove -y mysql*
sudo apt autoremove -y

# 关闭防火墙（生产环境可开放 3306 端口）
sudo ufw disable
```

### 2. 安装 MySQL 8.0（官方 APT 仓库）

bash

运行

```
# 1. 更新包索引
sudo apt update

# 2. 安装依赖
sudo apt install -y wget lsb-release

# 3. 下载官方 APT 配置包
wget https://dev.mysql.com/get/mysql-apt-config_0.8.33-1_all.deb

# 4. 安装配置包（默认选择 MySQL 8.0，直接 OK 即可）
sudo dpkg -i mysql-apt-config_0.8.33-1_all.deb

# 5. 再次更新包索引并安装
sudo apt update
sudo apt install -y mysql-server
```

### 3. 初始化与启动

bash

运行

```
# 安装过程中会提示设置 root 密码（直接设置，无需临时密码）
# 启动 MySQL 服务（默认已启动）
sudo systemctl start mysql
sudo systemctl enable mysql
```

### 4. 配置文件（面试常考）

编辑 `/etc/mysql/mysql.conf.d/mysqld.cnf`：

ini

```
[mysqld]
port=3306
user=mysql
datadir=/var/lib/mysql
socket=/var/run/mysqld/mysqld.sock

# 字符集（同 Fedora）
character-set-server=utf8mb4
collation-server=utf8mb4_0900_ai_ci

# 认证插件
default_authentication_plugin=mysql_native_password

# 日志
log-error=/var/log/mysql/error.log
pid-file=/var/run/mysqld/mysqld.pid
```

修改后重启：`sudo systemctl restart mysql`

### 5. 安全设置（同 Fedora）

bash

运行

```
sudo mysql_secure_installation
```

### 6. 远程连接配置（同 Fedora）

bash

运行

```
# 登录 MySQL 后执行用户创建和授权命令
# 配置文件中注释 bind-address = 127.0.0.1（或改为 0.0.0.0）
```

---

## 三、验证安装（双环境通用）

bash

运行

```
# 1. 登录 MySQL
mysql -u root -p

# 2. 查看版本
SELECT VERSION();

# 3. 查看字符集
SHOW VARIABLES LIKE 'character%';

# 4. 创建测试数据库
CREATE DATABASE test_db;
USE test_db;
CREATE TABLE test_table (id INT PRIMARY KEY, name VARCHAR(20));
INSERT INTO test_table VALUES (1, 'MySQL8.0');
SELECT * FROM test_table;
```