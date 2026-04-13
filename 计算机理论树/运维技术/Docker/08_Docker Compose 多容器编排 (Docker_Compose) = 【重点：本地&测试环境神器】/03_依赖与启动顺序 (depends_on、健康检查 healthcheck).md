
多容器应用的核心痛点之一是容器的启动顺序管理，例如后端服务必须等待数据库、缓存服务启动并就绪后才能启动，否则会出现连接失败、服务启动异常的问题。Compose 通过`depends_on`与`healthcheck`两大核心特性解决该问题。

### 3.1 depends_on 依赖声明

`depends_on`字段用于定义服务之间的启动依赖关系，决定服务的启动与停止顺序，核心作用有两点：

1. **启动顺序控制**：被依赖的服务先启动，当前服务后启动；
2. **停止顺序控制**：当前服务先停止，被依赖的服务后停止，避免停止过程中出现连接异常；
3. **网络依赖保障**：被依赖的服务先完成网络初始化，当前服务启动时可正常解析被依赖服务的域名。

#### 基础语法示例

```yaml
services:
  # 后端应用服务
  backend:
    build: ./backend
    depends_on:
      - mysql
      - redis
  # 数据库服务
  mysql:
    image: mysql:8.0
  # 缓存服务
  redis:
    image: redis:7.2-alpine
```

上述配置中，Compose 会按`mysql → redis → backend`的顺序启动服务，按`backend → redis → mysql`的顺序停止服务。

#### 3.1.1 核心局限性（高频踩坑点）

`depends_on`**仅能保证容器的启动顺序，无法保证容器内的应用程序就绪**。

例如：MySQL 容器启动后，容器状态为 Running，但 MySQL 数据库还在初始化、权限配置、数据恢复，此时后端服务启动会出现数据库连接失败的问题，这是新手最高频的踩坑点。

### 3.2 healthcheck 健康检查

健康检查是解决`depends_on`局限性的唯一官方方案，用于定义容器内应用程序的就绪状态检测规则，Docker 会定期执行检测命令，根据返回结果判断应用是否正常运行、是否就绪。

#### 3.2.1 核心语法与参数

```yaml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: 123456
      MYSQL_DATABASE: app_db
    # 健康检查配置
    healthcheck:
      # 健康检查执行的命令，必须返回0退出码表示健康，非0表示不健康
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p123456"]
      # 检查间隔时间，默认30s
      interval: 5s
      # 单次检查的超时时间，默认30s
      timeout: 3s
      # 启动后延迟检查时间，适配应用初始化耗时，默认0s
      start_period: 10s
      # 连续失败多少次后，标记为不健康，默认3次
      retries: 3
```

**test 命令语法规范**：

- 推荐使用`["CMD", "命令", "参数1", "参数2"]`的 Exec 格式，不会启动 Shell 环境，无兼容性问题；
- 也可使用`CMD-SHELL`格式：`test: "mysqladmin ping -h localhost -u root -p123456"`，会通过`/bin/sh -c`执行命令，支持 Shell 特性；
- 命令返回 0 退出码表示健康，1 表示不健康，2 为保留值，严禁使用。

#### 3.2.2 典型服务健康检查示例

| 服务类型       | 健康检查命令示例                                                                                           | 核心说明                                             |
| ---------- | -------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| HTTP 服务    | `test: ["CMD", "curl", "-f", "http://localhost:8080/health"]`                                      | 检测服务健康接口，返回 2xx 状态码表示健康，`-f`参数会在非 2xx 时返回非 0 退出码 |
| MySQL      | `test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]` | 使用 mysqladmin 工具检测数据库服务可用性                       |
| Redis      | `test: ["CMD", "redis-cli", "ping"]`                                                               | 使用 redis-cli 检测服务可用性，返回 PONG 表示健康                |
| PostgreSQL | `test: ["CMD", "pg_isready", "-U", "postgres", "-d", "app_db"]`                                    | 使用 pg_isready 工具检测数据库就绪状态                        |
| Nginx      | `test: ["CMD", "nginx", "-t"]`                                                                     | 检测 Nginx 配置有效性与服务运行状态                            |

### 3.3 依赖与就绪状态的完整解决方案

Compose V2 支持`depends_on`的`condition`条件配置，可结合`healthcheck`实现**等待依赖服务完全就绪后，再启动当前服务**，彻底解决启动顺序与应用就绪的问题，是生产环境的唯一标准方案。

#### 完整语法示例

```yaml
services:
  # 后端应用服务
  backend:
    build: ./backend
    depends_on:
      mysql:
        # 等待mysql服务的健康检查通过
        condition: service_healthy
      redis:
        # 等待redis服务启动完成（容器Running）
        condition: service_started
    # 自身的健康检查
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 5s
      timeout: 3s
      retries: 3
      start_period: 15s

  # 数据库服务
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: 123456
      MYSQL_DATABASE: app_db
    volumes:
      - mysql_data:/var/lib/mysql
    # 健康检查配置
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p123456"]
      interval: 5s
      timeout: 3s
      retries: 3
      start_period: 20s

  # 缓存服务
  redis:
    image: redis:7.2-alpine
    volumes:
      - redis_data:/data

volumes:
  mysql_data:
  redis_data:
```

#### condition 可选值说明

|条件值|核心规则|适用场景|
|---|---|---|
|`service_started`|仅等待依赖服务的容器启动完成（状态为 Running），默认值|无初始化耗时的轻量服务，如静态文件服务、无状态代理|
|`service_healthy`|等待依赖服务的健康检查通过，标记为 healthy 状态|数据库、缓存、后端服务等有初始化耗时的有状态服务，生产环境推荐|
|`service_completed_successfully`|等待依赖服务执行完成并正常退出（退出码为 0）|初始化任务、数据迁移、前置脚本等一次性任务服务|

### 生产级最佳实践

1. 所有有状态服务、后端业务服务必须配置`healthcheck`健康检查，明确应用的就绪状态；
2. 服务之间的依赖必须通过`condition: service_healthy`配置，严格等待依赖服务就绪后再启动，避免启动失败；
3. 健康检查的`interval`、`start_period`必须根据应用的实际初始化耗时合理配置，避免误判；
4. 健康检查命令必须轻量、快速执行，严禁使用耗时过长、资源占用过高的检测命令，避免影响容器性能。
