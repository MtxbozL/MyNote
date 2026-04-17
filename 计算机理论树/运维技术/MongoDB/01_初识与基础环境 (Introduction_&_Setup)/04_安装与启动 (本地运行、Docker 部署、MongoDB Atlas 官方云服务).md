
## 1.4 MongoDB 安装与启动

### 1.4.1 本地原生安装与启动

本地原生安装指将 MongoDB 二进制包直接部署在物理机 / 虚拟机操作系统中，适用于开发、测试与生产环境，可实现对数据库的完全控制。

1. **安装包获取**：从 MongoDB 官方下载中心获取对应操作系统、CPU 架构的长期支持稳定版（LTS），禁止将开发版（Dev）用于生产环境。
2. **系统级安装**：
    
    - Windows：通过 MSI 安装向导执行，可选择安装为系统服务实现开机自启；
    - Linux（RHEL/Debian 系列）：配置官方软件源，通过 yum/apt 包管理器安装，自动配置系统服务与环境变量；
    - macOS：通过 Homebrew 执行`brew install mongodb-community`完成安装。
    
3. **核心配置**：推荐使用 YAML 格式配置文件管理参数，核心必填项包括：
    
    - `dbpath`：数据文件存储目录，需保证运行用户拥有读写权限；
    - `logpath`：日志文件存储路径；
    - 核心可选参数：`port`（服务监听端口，默认 27017）、`bind_ip`（绑定 IP，默认仅监听 127.0.0.1，远程访问需修改）、`auth`（身份认证开关，生产环境必须开启）、`fork`（Linux/macOS 后台守护进程开关）。
    
4. **启动与验证**：
    
    - 启动命令：`mongod -f /path/to/mongod.conf`；
    - 系统服务启动：`systemctl start mongod`（Linux）；
    - 可用性验证：通过`mongosh`连接实例，执行`db.version()`，成功返回版本号即代表启动正常。
    

### 1.4.2 Docker 容器化部署

Docker 部署为轻量化容器化方案，适用于开发测试、多版本隔离、CI/CD 流水线场景，可实现开箱即用与环境一致性。

1. **镜像获取**：从 Docker Hub 拉取官方镜像，命令为`docker pull mongo:<version>`，推荐指定稳定版本号，禁止使用 latest 标签。
2. **核心启动命令**：

```bash
docker run -d \
  --name mongodb \
  -p 27017:27017 \
  -v /local/path/data:/data/db \
  -v /local/path/log:/var/log/mongodb \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=your_secure_password \
  mongo:<version> \
  --auth
```

3. **参数说明**：
    
    - `-d`：后台运行容器，`-p`：宿主机与容器的端口映射；
    - `-v`：数据卷挂载，实现数据持久化，避免容器删除后数据丢失；
    - `-e`：初始化 root 管理员账号密码，`--auth`：开启身份认证，保障容器安全。
    
4. **连接验证**：执行`docker exec -it mongodb mongosh -u admin -p your_secure_password`进入容器内 Shell，验证服务可用性。

### 1.4.3 MongoDB Atlas 官方云服务部署

MongoDB Atlas 是 MongoDB 官方提供的全托管 DBaaS 平台，适用于生产环境、初创团队、跨云部署场景，无需运维底层基础设施，开箱即用地提供企业级能力。

1. **账号注册**：通过 MongoDB Atlas 官网注册账号，支持个人免费版（M0 免费集群）与企业版付费套餐。
2. **集群创建**：选择云服务商（AWS/Azure/GCP）、部署区域、集群规格，免费版可创建单区域共享型副本集集群。
3. **安全配置**：配置 IP 访问白名单、创建数据库用户与对应权限，开启 TLS 加密传输。
4. **连接与使用**：获取集群连接字符串，通过 mongosh、Compass 或应用驱动完成连接，无需关注底层节点运维。Atlas 原生提供副本集高可用、自动备份、监控告警、性能优化等能力。
