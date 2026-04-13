
`docker-compose.yml`是 Compose 的核心配置文件，采用 YAML 语法编写，声明式定义多容器应用的所有资源，必须严格遵循 YAML 语法规范（缩进敏感、禁止使用 Tab 键）。

Compose 配置文件的顶级字段分为四大核心模块，完全对应目录内的规范，结构如下：

```yaml
# 顶级字段
version: # 配置文件语法版本（Compose Specification已非必填）
services: # 核心：服务定义，必选字段
networks: # 网络拓扑定义
volumes: # 全局数据卷定义
configs: # 配置文件管理（扩展）
secrets: # 敏感信息管理（扩展）
```

### 2.1 version 语法版本号

`version`字段用于声明 Compose 配置文件的语法版本，历史上分为 v1、v2、v3 三大版本系列：

- v1 版本：已完全废弃，无 version 顶级字段，不支持网络、数据卷等高级特性；
- v2 版本：针对单主机场景优化，支持健康检查、依赖顺序、资源限制等核心特性；
- v3 版本：兼容单主机与 Swarm 集群场景，移除了部分单主机特性，增加了集群编排相关配置。

**当前规范说明**：Docker 官方已推出统一的**Compose Specification**，替代了原有的版本号划分，`version`字段已非必填，新版 Docker Engine 与 Compose V2 会自动适配所有合规的配置语法。生产环境建议省略`version`字段，遵循最新的 Compose Specification 规范，无需绑定特定版本号。

### 2.2 services 服务定义（核心必选）

`services`是配置文件的核心必选字段，用于定义项目内的所有服务，每个子字段对应一个服务，服务名作为 DNS 解析名称，同一项目内的服务可通过服务名直接互相访问，对应第六章自定义网络的内置 DNS 解析机制。

本节覆盖服务定义的核心高频字段，所有字段均与前序章节的`docker run`参数一一对应，保证知识体系的连贯性。

#### 2.2.1 镜像与构建配置

|字段|核心作用|语法示例|生产规范|
|---|---|---|---|
|`image`|指定服务使用的镜像，与`docker run`的镜像参数完全一致，支持固定版本、摘要、私有仓库地址|`image: nginx:1.27.0-alpine`<br><br>`image: harbor.internal.example.com/library/mysql:8.0`|生产环境必须指定固定版本标签或摘要，严禁使用`latest`标签，避免版本漂移|
|`build`|定义基于 Dockerfile 构建镜像的规则，替代手动`docker build`，与`image`配合使用时，会将构建后的镜像打上`image`指定的标签|基础用法：<br><br>`build: ./backend`<br><br>完整用法：<br><br>`build:`<br><br>`context: ./backend`<br><br>`dockerfile: Dockerfile.prod`<br><br>`args:`<br><br>`JAVA_VERSION: 17`|生产环境必须显式指定`context`与`dockerfile`，构建参数通过`args`传递，对应第七章的`ARG`指令，严禁硬编码构建参数|
|`pull_policy`|定义镜像拉取策略|`pull_policy: always`|可选值：`always`（每次启动都拉取）、`never`（不拉取，仅使用本地镜像）、`if_not_present`（本地不存在时拉取，默认值）、`build`（优先构建，不拉取）|

#### 2.2.2 容器运行核心配置

|字段|核心作用|语法示例|对应 docker run 参数|
|---|---|---|---|
|`command`|覆盖镜像默认的`CMD`指令，定义容器启动执行的命令|`command: ["nginx", "-g", "daemon off;"]`<br><br>`command: java -jar app.jar`|`docker run`末尾的命令覆盖参数|
|`entrypoint`|覆盖镜像默认的`ENTRYPOINT`指令，定义容器入口程序|`entrypoint: ["/app/entrypoint.sh"]`|`--entrypoint`|
|`environment`|定义容器的环境变量，对应第七章的`ENV`指令，支持数组与键值对两种格式|键值对格式：<br><br>`environment:`<br><br>`MYSQL_ROOT_PASSWORD: 123456`<br><br>`TZ: Asia/Shanghai`<br><br>数组格式：<br><br>`environment:`<br><br>`- MYSQL_ROOT_PASSWORD=123456`|`-e/--env`|
|`env_file`|从外部文件加载环境变量，避免敏感信息硬编码在 yml 文件中|`env_file:`<br><br>`- .env`<br><br>`- ./config/mysql.env`|`--env-file`|
|`working_dir`|指定容器内的工作目录，覆盖镜像默认的`WORKDIR`指令|`working_dir: /app`|`-w/--workdir`|
|`user`|指定容器运行的用户，覆盖镜像默认的`USER`指令，生产环境降权运行必备|`user: 1000:1000`|`-u/--user`|
|`restart`|定义容器的重启策略，对应第四章的`--restart`参数，生产环境高可用必备|`restart: unless-stopped`|`--restart`|
|`extra_hosts`|向容器的`/etc/hosts`文件添加主机名与 IP 映射，解决自定义域名解析需求|`extra_hosts:`<br><br>`- "api.example.com:192.168.1.100"`|`--add-host`|

#### 2.2.3 端口映射配置

`ports`字段定义宿主机到容器的端口映射规则，完全对应第四章的`-p/--publish`参数，支持两种语法格式：

1. 简写格式（最常用）：

    ```yaml
    ports:
      - "8080:80" # 宿主机8080端口映射到容器80端口
      - "192.168.1.100:3306:3306" # 绑定指定IP
      - "53:53/udp" # 指定UDP协议
    ```
    
2. 完整格式（生产环境推荐，无歧义）：

    ```yaml
    ports:
      - target: 80 # 容器内端口
        published: 8080 # 宿主机端口
        protocol: tcp # 协议
        host_ip: 192.168.1.100 # 绑定的宿主机IP
        mode: host # 端口模式，host为主机模式，ingress为Swarm集群模式
    ```
    

**生产规范**：

- 必须显式指定绑定的宿主机内网 IP，严禁使用默认的`0.0.0.0`全网绑定，缩小攻击面；
- 数据库、缓存等内部服务严禁对外暴露端口，仅通过内部网络与其他服务互通；
- 端口映射必须严格遵循最小权限原则，仅暴露必须对外提供服务的端口。

#### 2.2.4 数据卷与挂载配置

`volumes`字段定义容器的持久化挂载配置，完全对应第五章的三种挂载方案，支持绑定挂载、命名卷、tmpfs 挂载，语法格式与`docker run`的`-v/--mount`参数完全对应。

核心语法示例：

```yaml
services:
  mysql:
    image: mysql:8.0
    volumes:
      # 1. 命名卷挂载（生产环境推荐，对应全局volumes字段定义）
      - mysql_data:/var/lib/mysql
      # 2. 绑定挂载（仅开发环境/配置文件挂载使用）
      - ./mysql/my.cnf:/etc/mysql/my.cnf:ro
      # 3. tmpfs内存挂载（临时数据/敏感数据）
      - type: tmpfs
        target: /tmp
        tmpfs:
          size: 100m
          mode: 1777
volumes:
  # 全局命名卷定义
  mysql_data:
```

**生产规范**：

- 业务核心数据必须使用**项目级命名卷**，严禁使用匿名卷与绑定挂载，保证数据生命周期独立于容器，便于统一管理；
- 配置文件挂载优先使用`ro`只读模式，避免容器内进程误修改配置；
- 绑定挂载的宿主机路径必须使用相对路径，保证项目的可移植性，严禁使用绝对路径。

#### 2.2.5 网络配置

`networks`字段定义服务接入的网络，对应第六章的自定义网络，服务必须接入同一网络才能互相通信，未显式指定时，Compose 会自动将服务接入项目默认网络。

语法示例：

```yaml
services:
  nginx:
    image: nginx:1.27.0-alpine
    # 接入两个网络
    networks:
      - frontend_net
      - backend_net
  mysql:
    image: mysql:8.0
    # 仅接入后端网络，与前端网络隔离
    networks:
      - backend_net
networks:
  frontend_net:
  backend_net:
```

### 2.3 networks 网络拓扑定义

`networks`顶级字段用于定义项目内的所有自定义网络，对应第六章的`docker network create`命令，未显式定义时，Compose 会自动创建一个名为`[项目名]_default`的默认 bridge 网络，项目内所有服务默认接入该网络。

#### 核心配置语法

```yaml
networks:
  # 基础自定义网络，使用默认bridge驱动与配置
  frontend_net:
    # 网络驱动，单主机环境默认bridge，集群环境使用overlay
    driver: bridge
    # 网络标签，用于分类管理与过滤
    labels:
      env: production
      business: frontend
  # 高级自定义网络，显式指定网段、网关等配置
  backend_net:
    driver: bridge
    # IP地址管理配置
    ipam:
      driver: default
      config:
        - subnet: 172.20.0.0/16
          gateway: 172.20.0.1
  # 引用外部已存在的网络，实现跨项目服务互通
  external_net:
    external: true
    name: public_proxy_net
```

#### 核心特性与生产规范

1. **项目级网络隔离**：Compose 为每个项目创建独立的网络，不同项目的服务默认无法互相通信，如需互通，需通过`external: true`引用公共网络；
2. **内置 DNS 解析**：同一网络内的服务，可直接通过服务名互相访问，无需硬编码 IP 地址，完美适配容器的动态生命周期；
3. **生产环境网络隔离规范**：必须按业务安全等级划分独立网络，如前端网络、后端业务网络、数据库网络，仅必要的服务接入多个网络实现跨网通信，严格遵循最小权限原则，缩小攻击面；
4. **网段规划**：生产环境必须显式指定`subnet`与`gateway`，提前完成 IP 地址规划，避免与宿主机、机房内网、云平台网段冲突。

### 2.4 volumes 全局数据卷定义

`volumes`顶级字段用于定义项目级的命名数据卷，对应第五章的`docker volume create`命令，服务通过卷名引用这些全局卷，实现多服务之间的数据共享与持久化。

#### 核心配置语法

```yaml
volumes:
  # 基础命名卷，使用默认本地驱动
  mysql_data:
    labels:
      env: production
      business: mysql
  # 引用外部已存在的命名卷，实现跨项目数据共享
  static_data:
    external: true
    name: public_static_data
  # 自定义驱动配置，如对接NFS分布式存储
  nfs_data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.200,rw
      device: ":/data/nfs/docker"
```

#### 核心特性与生产规范

1. **生命周期独立**：全局定义的命名卷生命周期完全独立于项目与容器，执行`docker compose down`时，默认不会删除命名卷，仅当添加`-v`参数时才会删除，彻底避免数据误删；
2. **多服务共享**：同一项目内的多个服务可同时挂载同一个全局命名卷，实现安全的数据共享；
3. **生产规范**：
    
    - 所有业务持久化数据必须使用全局命名卷，严禁使用匿名卷；
    - 核心业务数据卷必须添加标签，便于备份、迁移与管理；
    - 执行`docker compose down`时，严禁随意添加`-v`参数，否则会永久删除所有命名卷，导致业务数据丢失。
    
