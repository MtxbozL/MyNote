本文档覆盖 MySQL 官方二进制包（Generic Linux）在 Ubuntu 24.04 上的**全手动安装流程**，包含环境准备、依赖安装、目录配置、初始化、服务管理、安全配置全流程，适合需要灵活控制版本、自定义安装路径的专业场景。

---

## 前置核心说明

### 1. 二进制安装 vs 包管理器安装对比

表格

|维度|二进制安装（本文档）|apt 包管理器安装|
|---|---|---|
|版本灵活性|可自由选择任意官方版本（包括最新稳定版、历史版本）|仅能安装 Ubuntu 源内固定版本|
|安装路径|完全自定义（默认推荐 `/usr/local/mysql`）|系统默认路径分散|
|配置灵活性|可深度定制编译参数、目录结构|配置受限|
|维护难度|需手动管理升级、依赖、服务|一键升级，自动处理依赖|
|适用场景|生产环境特定版本需求、深度定制|日常开发、快速部署|

### 2. 环境要求

- Ubuntu 24.04 LTS（x86_64/arm64 架构均可，本文以 x86_64 为例）
- 至少 2GB 可用磁盘空间
- root 权限或 sudo 权限
- 确保系统未预装 MySQL/MariaDB（预装需先卸载，避免冲突）

---

## 一、环境准备与依赖安装

### 1. 卸载系统预装的 MySQL/MariaDB（避免冲突）

bash

运行

```bash
# 1. 检查是否预装
dpkg -l | grep -E "mysql|mariadb"

# 2. 若有预装，完全卸载（谨慎操作，确认无重要数据）
sudo apt purge -y mysql-server mysql-client mysql-common mariadb-server mariadb-client
sudo apt autoremove -y
sudo rm -rf /etc/mysql /var/lib/mysql /var/log/mysql
```

### 2. 安装必要依赖包

Ubuntu 24.04 需安装以下依赖，否则 MySQL 无法启动：

bash

运行

```
sudo apt update && sudo apt install -y libaio1 libnuma-dev libncurses5-dev libtirpc-dev
```

---

## 二、下载 MySQL 官方二进制包

### 1. 选择并下载对应版本

访问 [MySQL 官方下载页](https://dev.mysql.com/downloads/mysql/)，选择：

- **操作系统**：Linux - Generic
- **架构**：x86_64（或对应你的服务器架构）
- **版本**：推荐最新稳定版（如 MySQL 8.4 LTS）

本文以 **MySQL 8.4.2 LTS（x86_64）** 为例，直接在终端下载：

bash

运行

```
# 进入临时下载目录
cd /tmp

# 下载 MySQL 8.4.2 二进制包（若版本更新，替换为最新链接）
wget https://dev.mysql.com/get/Downloads/MySQL-8.4/mysql-8.4.2-linux-glibc2.28-x86_64.tar.xz

# 验证文件完整性（可选，对比官方SHA256校验值）
sha256sum mysql-8.4.2-linux-glibc2.28-x86_64.tar.xz
```

---

## 三、创建用户与目录结构

### 1. 创建 mysql 专用用户和用户组

MySQL 需以非 root 用户运行，提升安全性：

bash

运行

```
# 创建 mysql 用户组
sudo groupadd mysql

# 创建 mysql 用户（不允许登录系统，仅用于运行服务）
sudo useradd -r -s /sbin/nologin -g mysql mysql
```

### 2. 规划并创建安装目录与数据目录

推荐目录结构（可自定义，需同步修改后续配置）：

表格

|目录类型|路径|说明|
|---|---|---|
|安装目录|`/usr/local/mysql`|MySQL 主程序目录|
|数据目录|`/data/mysql`|数据库文件存储目录（生产环境建议单独挂载磁盘）|
|日志目录|`/var/log/mysql`|错误日志、慢查询日志目录|
|套接字目录|`/var/run/mysqld`|MySQL socket 文件目录|

执行命令创建目录并设置权限：

bash

运行

```
# 1. 创建所有必要目录
sudo mkdir -p /usr/local/mysql /data/mysql /var/log/mysql /var/run/mysqld

# 2. 设置目录所有者为 mysql 用户
sudo chown -R mysql:mysql /usr/local/mysql /data/mysql /var/log/mysql /var/run/mysqld

# 3. 设置目录权限（安全加固）
sudo chmod 750 /usr/local/mysql /data/mysql /var/log/mysql /var/run/mysqld
```

---

## 四、解压并配置 MySQL

### 1. 解压二进制包并移动到安装目录

bash

运行

```
# 1. 解压下载的 tar.xz 包（解压时间较长，耐心等待）
cd /tmp
sudo tar -xvf mysql-8.4.2-linux-glibc2.28-x86_64.tar.xz

# 2. 将解压后的文件移动到安装目录
sudo mv mysql-8.4.2-linux-glibc2.28-x86_64/* /usr/local/mysql/

# 3. 验证文件完整性
ls -la /usr/local/mysql/
```

### 2. 配置环境变量（可选，推荐）

将 MySQL 命令加入系统 PATH，方便直接使用 `mysql`、`mysqld` 等命令：

bash

运行

```
# 编辑全局环境变量文件
sudo vi /etc/profile.d/mysql.sh
```

写入以下内容：

bash

运行

```
export MYSQL_HOME=/usr/local/mysql
export PATH=$MYSQL_HOME/bin:$MYSQL_HOME/lib:$PATH
```

保存退出后，使环境变量立即生效：

bash

运行

```
sudo chmod +x /etc/profile.d/mysql.sh
source /etc/profile.d/mysql.sh
```

验证：执行 `mysql --version`，能正常输出版本号说明配置成功。

---

## 五、创建并编辑 MySQL 配置文件 my.cnf

Ubuntu 24.04 二进制安装需手动创建配置文件，路径为 `/etc/my.cnf`（全局配置）：

bash

运行

```
sudo vi /etc/my.cnf
```

写入以下配置（根据你的服务器资源调整参数，生产环境需优化）：

ini

```
[mysqld]
# 基础配置
basedir=/usr/local/mysql          # 安装目录
datadir=/data/mysql                # 数据目录（核心！必须和之前创建的一致）
socket=/var/run/mysqld/mysqld.sock # Socket文件路径
port=3306                          # 默认端口
pid-file=/var/run/mysqld/mysqld.pid # PID文件路径
user=mysql                         # 运行用户（必须为mysql）

# 字符集配置（统一使用utf8mb4，避免乱码）
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci
init_connect='SET NAMES utf8mb4'

# 日志配置
log-error=/var/log/mysql/error.log # 错误日志路径
slow_query_log=1                    # 开启慢查询日志
slow_query_log_file=/var/log/mysql/slow.log
long_query_time=2                   # 慢查询阈值（秒）

# 性能配置（根据服务器内存调整，示例为2GB内存配置）
innodb_buffer_pool_size=1G          # InnoDB缓冲池大小（推荐为内存的50%-70%）
innodb_log_file_size=256M           # 日志文件大小
max_connections=200                 # 最大连接数

[mysql]
# 客户端配置
default-character-set=utf8mb4
socket=/var/run/mysqld/mysqld.sock

[client]
socket=/var/run/mysqld/mysqld.sock
```

保存退出后，设置配置文件权限：

bash

运行

```
sudo chown mysql:mysql /etc/my.cnf
sudo chmod 640 /etc/my.cnf
```

---

## 六、初始化 MySQL 数据库（核心步骤）

### 1. 执行初始化命令

⚠️ 核心注意：初始化会生成**临时 root 密码**，必须保存！

bash

运行

```
# 进入 MySQL 安装目录的 bin 目录
cd /usr/local/mysql/bin

# 执行初始化（--initialize-insecure 可设置空密码，生产环境不推荐）
sudo ./mysqld --initialize --user=mysql --basedir=/usr/local/mysql --datadir=/data/mysql
```

### 2. 获取临时 root 密码

初始化完成后，临时密码会写入错误日志，执行以下命令查看：

bash

运行

```
sudo grep 'temporary password' /var/log/mysql/error.log
```

示例输出（记住最后面的临时密码，如 `abc123!@#`）：

plaintext

```
2024-05-20T10:00:00.000000Z 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: abc123!@#
```

---

## 七、启动 MySQL 服务并配置开机自启

### 1. 手动启动 MySQL（测试用）

bash

运行

```
# 方式1：使用 mysqld_safe 启动（推荐，自带崩溃重启保护）
cd /usr/local/mysql/bin
sudo ./mysqld_safe --user=mysql &

# 方式2：直接使用 mysqld 启动（不推荐，生产环境用方式1）
# sudo ./mysqld --user=mysql &
```

### 2. 验证服务是否启动成功

bash

运行

```
# 1. 查看进程是否存在
ps -ef | grep mysqld

# 2. 查看端口是否监听
sudo ss -tulpn | grep :3306

# 3. 查看错误日志，确认无报错
sudo tail -f /var/log/mysql/error.log
```

✅ 正常：能看到 mysqld 进程，3306 端口处于 LISTEN 状态，错误日志无 ERROR 级报错。

### 3. 配置 systemd 服务（生产环境必备，实现开机自启）

手动创建 systemd 服务文件，方便用 `systemctl` 管理服务：

bash

运行

```
sudo vi /etc/systemd/system/mysqld.service
```

写入以下内容：

ini

```
[Unit]
Description=MySQL Server
After=network.target syslog.target

[Service]
Type=forking
User=mysql
Group=mysql
ExecStart=/usr/local/mysql/bin/mysqld_safe --defaults-file=/etc/my.cnf
ExecStop=/usr/local/mysql/bin/mysqladmin -S /var/run/mysqld/mysqld.sock -u root -p shutdown
PIDFile=/var/run/mysqld/mysqld.pid
Restart=on-failure
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

保存退出后，重载 systemd 并设置开机自启：

bash

运行

```
# 1. 重载 systemd 配置
sudo systemctl daemon-reload

# 2. 停止之前手动启动的 MySQL（如果有）
sudo /usr/local/mysql/bin/mysqladmin -S /var/run/mysqld/mysqld.sock -u root -p shutdown

# 3. 用 systemctl 启动 MySQL
sudo systemctl start mysqld

# 4. 设置开机自启
sudo systemctl enable mysqld

# 5. 验证服务状态
sudo systemctl status mysqld
```

✅ 正常：显示 `active (running)`，且无报错。

---

## 八、安全配置（初始化后必须执行）

### 1. 登录 MySQL 并修改临时 root 密码

bash

运行

```
# 用临时密码登录（输入刚才保存的临时密码）
mysql -u root -p
```

登录成功后，执行以下 SQL 修改 root 密码（替换为你的强密码）：

sql

```
-- 修改 root 本地登录密码（生产环境必须用强密码）
ALTER USER 'root'@'localhost' IDENTIFIED BY 'YourStrongPassword@123';

-- 刷新权限
FLUSH PRIVILEGES;

-- 退出 MySQL
EXIT;
```

### 2. 执行 MySQL 安全加固脚本（推荐）

MySQL 自带安全加固脚本，可删除匿名用户、禁止 root 远程登录、删除测试数据库：

bash

运行

```
sudo /usr/local/mysql/bin/mysql_secure_installation
```

按提示操作（全部选 `y` 即可，除了 “修改 root 密码” 若已修改可选 `n`）。

---

## 九、验证安装与常用操作

### 1. 验证 MySQL 版本与基本功能

bash

运行

```
# 1. 登录 MySQL（用新密码）
mysql -u root -p

-- 2. 查看版本
SELECT VERSION();

-- 3. 查看当前数据库
SHOW DATABASES;

-- 4. 创建测试数据库（验证读写权限）
CREATE DATABASE test_db;
USE test_db;
CREATE TABLE test_table (id INT, name VARCHAR(20));
INSERT INTO test_table VALUES (1, 'test');
SELECT * FROM test_table;

-- 5. 清理测试数据
DROP DATABASE test_db;

-- 6. 退出
EXIT;
```

### 2. 配置防火墙放行 3306 端口（如需远程连接）

⚠️ 生产环境不建议直接开放 3306 端口到公网，推荐通过 VPN 或跳板机连接！

bash

运行

```
# 1. 放行 3306 端口
sudo ufw allow 3306/tcp

# 2. 重载防火墙
sudo ufw reload

# 3. 验证规则
sudo ufw status
```

### 3. 开启 root 远程登录（可选，不推荐生产环境使用）

如需远程连接 MySQL，需创建允许远程登录的用户（**不要直接用 root 远程登录**）：

sql

```
-- 登录 MySQL 后执行
-- 创建允许远程登录的用户（替换用户名、密码、允许的IP）
CREATE USER 'remote_user'@'%' IDENTIFIED BY 'RemotePassword@123';

-- 授予所有权限（按需授权，生产环境最小权限原则）
GRANT ALL PRIVILEGES ON *.* TO 'remote_user'@'%' WITH GRANT OPTION;

-- 刷新权限
FLUSH PRIVILEGES;
```

---

## 十、常见问题排查

### 1. 初始化失败

- 检查数据目录 `/data/mysql` 是否为空，非空需先清空
- 检查目录权限是否为 `mysql:mysql`
- 查看错误日志 `/var/log/mysql/error.log` 具体报错

### 2. 服务启动失败

- 检查配置文件 `/etc/my.cnf` 路径是否正确
- 检查端口 3306 是否被占用：`sudo ss -tulpn | grep :3306`
- 查看错误日志定位具体原因

### 3. 忘记 root 密码

bash

运行

```
# 1. 停止 MySQL 服务
sudo systemctl stop mysqld

# 2. 跳过权限验证启动
sudo /usr/local/mysql/bin/mysqld_safe --skip-grant-tables --user=mysql &

# 3. 无密码登录
mysql -u root

-- 4. 修改密码
USE mysql;
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'NewPassword@123';
FLUSH PRIVILEGES;
EXIT;

# 5. 重启 MySQL 服务
sudo systemctl restart mysqld
```

---

## 十一、卸载二进制安装的 MySQL（如需）

⚠️ 警告：卸载会删除所有数据库数据，务必提前备份！

bash

运行

```
# 1. 停止 MySQL 服务
sudo systemctl stop mysqld
sudo systemctl disable mysqld

# 2. 删除 systemd 服务文件
sudo rm -f /etc/systemd/system/mysqld.service
sudo systemctl daemon-reload

# 3. 删除所有 MySQL 相关目录（谨慎操作）
sudo rm -rf /usr/local/mysql /data/mysql /var/log/mysql /var/run/mysqld /etc/my.cnf

# 4. 删除 mysql 用户和用户组
sudo userdel -r mysql
sudo groupdel mysql

# 5. 删除环境变量文件
sudo rm -f /etc/profile.d/mysql.sh
```