
软件管理模块是 Ansible 批量管理被控节点软件包的核心工具，覆盖主流 Linux 发行版的系统包管理、Python 包管理、文件下载等场景，所有模块均**内置完整幂等性**。

### 3.4.1 yum 模块

**核心定义**：用于 RHEL、CentOS、Rocky Linux、AlmaLinux、Fedora 等 RPM 系发行版的软件包管理，支持安装、升级、降级、卸载、软件源管理等全功能操作。

#### 核心参数

|参数|核心作用|必填项|
|:--|:--|:--|
|`name`|软件包名称，支持版本号、列表格式、本地 rpm 路径、远程 URL|是|
|`state`|操作状态，可选 present/installed、latest、absent/removed、downgrade|否|
|`enablerepo`|执行操作时临时启用的软件源|否|
|`disablerepo`|执行操作时临时禁用的软件源|否|
|`update_cache`|执行前更新软件源缓存，可选 yes/no/only（仅更新缓存）|否|
|`lock_timeout`|yum 锁等待超时时间，单位秒，默认 30|否|

#### 标准示例

```bash
# 1. 安装nginx，确保已安装，不升级现有版本
ansible all -m yum -a "name=nginx state=present" -b

# 2. 批量安装多个软件包
ansible all -m yum -a "name=['nginx', 'mysql', 'redis', 'git', 'htop'] state=present" -b

# 3. 安装指定版本的软件包
ansible all -m yum -a "name=nginx-1.24.0 state=present" -b

# 4. 升级nginx到软件源最新版本
ansible all -m yum -a "name=nginx state=latest" -b

# 5. 卸载软件包
ansible all -m yum -a "name=nginx state=absent" -b

# 6. 临时启用epel源安装软件
ansible all -m yum -a "name=htop state=present enablerepo=epel" -b

# 7. 仅更新软件源缓存
ansible all -m yum -a "update_cache=only" -b
```

#### 核心注意事项

1. 内置幂等性，仅当软件包状态与目标状态不一致时才执行操作，多次执行无副作用；
2. 必须通过`-b`参数提权执行，否则权限不足；
3. 生产环境推荐指定固定版本号，避免使用`state=latest`，防止非预期的版本升级导致环境不一致；
4. 支持本地 rpm 文件与远程 http/https rpm 包的安装。

### 3.4.2 apt 模块

**核心定义**：用于 Debian、Ubuntu 等 Debian 系发行版的软件包管理，功能与 yum 模块对等，适配 apt 包管理体系。

#### 核心参数

|参数|核心作用|必填项|
|:--|:--|:--|
|`name`|软件包名称，支持版本号、列表格式|是|
|`state`|操作状态，可选 present/installed、latest、absent/removed|否|
|`update_cache`|执行前更新 apt 缓存，可选 yes/no/only|否|
|`cache_valid_time`|缓存有效时间，单位秒，仅当缓存超期才更新|否|
|`install_recommends`|是否安装推荐依赖包，默认 yes|否|
|`purge`|卸载时是否同时删除配置文件，默认 no|否|

#### 标准示例

```bash
# 1. 安装nginx
ansible all -m apt -a "name=nginx state=present" -b

# 2. 批量安装多个软件包
ansible all -m apt -a "name=['nginx', 'mysql-server', 'redis', 'git'] state=present" -b

# 3. 彻底卸载软件包与配置文件
ansible all -m apt -a "name=nginx state=absent purge=yes" -b

# 4. 更新apt缓存，有效期1小时
ansible all -m apt -a "update_cache=yes cache_valid_time=3600" -b
```

#### 核心注意事项

1. 必须提权执行，内置幂等性，与 yum 模块特性一致；
2. `purge=yes`会彻底删除软件包与配置文件，谨慎使用；
3. 生产环境同样推荐指定固定版本号，避免非预期升级。

### 3.4.3 pip 模块

**核心定义**：用于 Python 包的安装、升级、卸载管理，支持全局环境、虚拟环境、多 Python 版本适配。

#### 核心参数

|参数|核心作用|必填项|
|:--|:--|:--|
|`name`|Python 包名称，支持版本号、列表格式、git 仓库地址|是|
|`state`|操作状态，可选 present/installed、latest、absent|否|
|`virtualenv`|指定虚拟环境路径，在该环境中安装包|否|
|`executable`|指定 pip 可执行文件路径，适配多 Python 版本|否|
|`requirements`|requirements.txt 依赖文件路径|否|
|`extra_args`|传递给 pip 的额外参数，如指定镜像源|否|

#### 标准示例

```bash
# 1. 全局安装requests包
ansible all -m pip -a "name=requests state=present" -b

# 2. 安装指定版本的包
ansible all -m pip -a "name=requests==2.31.0 state=present"

# 3. 批量安装requirements.txt依赖
ansible all -m pip -a "requirements=/opt/app/requirements.txt"

# 4. 指定国内镜像源安装
ansible all -m pip -a "name=django extra_args='--index-url https://pypi.tuna.tsinghua.edu.cn/simple'"

# 5. 在指定虚拟环境中安装包
ansible all -m pip -a "name=flask virtualenv=/opt/app/venv"
```

#### 核心注意事项

1. 前置依赖：被控节点必须安装对应版本的 Python 与 pip；
2. 内置幂等性，仅当包状态与目标不一致时才执行操作；
3. 全局安装需要提权，虚拟环境安装无需提权，只需对应目录的读写权限。

### 3.4.4 get_url 模块

**核心定义**：用于从 HTTP/HTTPS/FTP 等 URL 下载文件到被控节点本地，支持断点续传、校验和验证、身份认证、代理等功能，是远程文件下载的核心模块。

#### 核心参数

|参数|核心作用|必填项|
|:--|:--|:--|
|`url`|文件的下载 URL，支持 HTTP/HTTPS/FTP|是|
|`dest`|被控节点的目标路径（文件 / 目录）|是|
|`mode/owner/group`|同 file 模块，设置下载文件的权限、所有者|否|
|`checksum`|文件校验和，格式为 <算法>:< 校验和 >，如 sha256:xxx，下载后校验|否|
|`timeout`|下载超时时间，单位秒，默认 10|否|
|`url_username/url_password`|HTTP 基本认证的用户名与密码|否|
|`validate_certs`|是否校验 HTTPS 证书，默认 yes|否|
|`force`|是否强制下载覆盖，默认 no，仅当文件不存在时下载|否|

#### 标准示例

```bash
# 1. 基础下载nginx源码包到被控节点
ansible all -m get_url -a "url=https://nginx.org/download/nginx-1.24.0.tar.gz dest=/opt/src/"

# 2. 下载文件并设置权限，校验sha256校验和
ansible all -m get_url -a "url=https://nginx.org/download/nginx-1.24.0.tar.gz dest=/opt/src/nginx-1.24.0.tar.gz mode=0644 checksum=sha256:58bed771f96a65c41f5c535c37a0f44d7a0d3e0d3d5e5d5d5d5d5d5d5d5d5d"

# 3. 强制重新下载文件
ansible all -m get_url -a "url=https://example.com/file.tar.gz dest=/opt/src/ force=yes"
```

#### 核心注意事项

1. 内置幂等性，默认仅当目标文件不存在时才下载，配合`checksum`参数可实现校验和变化时自动重新下载；
2. 支持断点续传，下载中断后重新执行会继续传输剩余部分；
3. 生产环境推荐使用`checksum`参数，确保下载文件的完整性与安全性，防止文件被篡改。

---
