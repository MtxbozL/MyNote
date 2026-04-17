## 🔍 错误根因

这是 Ubuntu 系统上 PostgreSQL 最常见的错误，**核心原因是 PostgreSQL 服务没有正常运行**，导致 `/var/run/postgresql/.s.PGSQL.5432` 这个 Unix 域套接字文件没有被创建。

---

## ✅ 分步排查解决（按顺序执行）

### 1. 第一步：检查 PostgreSQL 服务状态

```bash
sudo systemctl status postgresql
```

- 如果显示 `inactive (dead)` → 服务没启动
- 如果显示 `failed` → 服务启动失败
- 如果显示 `active (running)` → 服务正常，跳转到步骤 5

---

### 2. 第二步：启动 PostgreSQL 服务

```bash
# 启动服务
sudo systemctl start postgresql

# 设置开机自启（推荐）
sudo systemctl enable postgresql
```

启动后再次检查状态：

```bash
sudo systemctl status postgresql
```

如果显示 `active (running)`，问题解决，直接执行 `createdb testdb` 即可。

---

### 3. 第三步：如果服务启动失败，查看错误日志

这是最关键的一步，能定位具体失败原因：

```bash
# 查看 PostgreSQL 最新日志
sudo journalctl -u postgresql -f
```

常见失败原因及解决：

|常见错误|解决方法|
|---|---|
|端口 5432 被占用|杀死占用进程：`sudo lsof -i :5432` 找到 PID，然后 `sudo kill -9 PID`|
|数据目录权限错误|修复权限：`sudo chown -R postgres:postgres /var/lib/postgresql/`|
|配置文件错误|检查 `/etc/postgresql/[版本号]/main/postgresql.conf` 语法|

---

### 4. 第四步：如果服务根本没安装

如果执行 `systemctl status postgresql` 提示 `Unit postgresql.service could not be found`，说明你还没安装 PostgreSQL：

```bash
# 更新软件源
sudo apt update

# 安装 PostgreSQL 及客户端
sudo apt install postgresql postgresql-contrib
```

安装完成后，服务会自动启动，然后就可以创建数据库了。

---

### 5. 第五步：验证连接并创建数据库

服务正常运行后，**不要直接用 sudo createdb**，推荐用标准方式：

```bash
# 切换到 postgres 超级用户
sudo -u postgres psql

# 在 psql 终端里创建数据库
CREATE DATABASE testdb;

# 退出 psql
\q
```

或者直接用 createdb 命令（推荐）：

```bash
sudo -u postgres createdb testdb
```

---

## 💡 额外避坑提示

1. **不要用 `sudo createdb testdb`**：这会尝试用当前系统用户（ubuntu）创建数据库，而默认情况下 PostgreSQL 没有这个用户，会报 `role "ubuntu" does not exist` 错误。
2. **创建自己的数据库用户**（方便后续 FastAPI 连接）：
    
    ```bash
    sudo -u postgres createuser --interactive --pwprompt
    ```
    
    按照提示输入用户名和密码，然后给用户授权数据库：
    
    ```bash
    sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE testdb TO 你的用户名;"
    ```
    
3. **查看 PostgreSQL 版本**：
    
    ```bash
    psql --version
    ```
    
    不同版本的配置文件路径略有不同，比如 16 版本是 `/etc/postgresql/16/main/`。

---

## 🚀 快速验证

执行以下命令，如果能正常进入 psql 终端，说明一切正常：

```bash
sudo -u postgres psql testdb
```

需要我帮你写一个 FastAPI 连接 PostgreSQL 的完整配置示例吗？

帮我把这个问题整理成笔记

已完成思考

# PostgreSQL 常见错误笔记：Unix 套接字连接失败

## 一、错误现象

### 关键报错信息

```bash
createdb: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: No such file or directory
```

### 触发场景

在 Ubuntu 系统执行 `sudo createdb testdb` 或任何 PostgreSQL 客户端连接命令时，直接报错无法连接。

## 二、核心错误原因

**PostgreSQL 数据库服务没有正常运行**，导致系统没有创建 `/var/run/postgresql/.s.PGSQL.5432` 这个 Unix 域套接字文件，客户端无法通过默认方式连接到数据库。

## 三、分步排查解决（按优先级执行）

### 1. 第一步：检查服务状态（必做）

```bash
sudo systemctl status postgresql
```

- ✅ 正常结果：显示 `active (running)`
- ❌ 异常结果：显示 `inactive (dead)`（未启动）或 `failed`（启动失败）
- ⚠️ 特殊结果：提示 `Unit postgresql.service could not be found`（未安装）

---

### 2. 第二步：启动并设置开机自启（最常见解决方法）

```bash
# 立即启动 PostgreSQL 服务
sudo systemctl start postgresql

# 设置开机自动启动（推荐，避免重启服务器后重复操作）
sudo systemctl enable postgresql
```

启动后再次执行 `sudo systemctl status postgresql` 确认状态为 `active (running)`。

---

### 3. 第三步：服务启动失败？查看错误日志定位问题

如果服务显示 `failed`，执行以下命令查看详细错误日志：

```bash
# 实时查看最新日志
sudo journalctl -u postgresql -f
```

#### 常见启动失败原因及解决方案

|错误原因|解决命令|
|---|---|
|端口 5432 被其他进程占用|`sudo lsof -i :5432` 找到占用进程 PID<br><br>`sudo kill -9 进程PID` 杀死进程|
|数据目录权限错误|`sudo chown -R postgres:postgres /var/lib/postgresql/`|
|配置文件语法错误|检查 `/etc/postgresql/[版本号]/main/postgresql.conf`|

---

### 4. 第四步：服务根本没安装？执行安装命令

如果第一步提示服务不存在，说明系统未安装 PostgreSQL：

```bash
# 更新软件源
sudo apt update

# 安装 PostgreSQL 核心服务 + 常用工具
sudo apt install postgresql postgresql-contrib
```

安装完成后服务会自动启动，无需手动执行启动命令。

---

### 5. 第五步：验证并创建数据库（正确姿势）

⚠️ **不要直接用 `sudo createdb testdb`**（会报 `role "ubuntu" does not exist` 错误）

#### 方法一：通过 psql 终端创建

```bash
# 切换到 PostgreSQL 默认超级用户 postgres
sudo -u postgres psql

# 在 psql 终端内执行创建数据库命令
CREATE DATABASE testdb;

# 退出 psql
\q
```

#### 方法二：直接用 createdb 命令（推荐）

```bash
sudo -u postgres createdb testdb
```

## 四、避坑指南（FastAPI 开发必备）

1. **创建专属数据库用户（避免用 postgres 超级用户开发）**
    
    ```bash
    # 创建交互式用户（按提示输入用户名和密码）
    sudo -u postgres createuser --interactive --pwprompt
    
    # 给用户授予数据库全部权限
    sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE testdb TO 你的用户名;"
    ```
    
2. **FastAPI 连接 PostgreSQL 参考格式**
    
    ```python
    # SQLAlchemy 连接字符串
    SQLALCHEMY_DATABASE_URL = "postgresql://用户名:密码@localhost:5432/testdb"
    ```
    
3. **查看 PostgreSQL 版本**
    
    ```bash
    psql --version
    ```
    
    不同版本的配置文件路径略有不同，例如 16 版本为 `/etc/postgresql/16/main/`

## 五、快速验证

执行以下命令，如果能正常进入 testdb 数据库的 psql 终端，说明问题完全解决：

```bash
sudo -u postgres psql testdb
```