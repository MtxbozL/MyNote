
本节基于目录要求，实现完整的全栈微服务应用一键部署，覆盖前端反向代理、后端业务服务、缓存服务、数据库服务四大核心模块，严格遵循前序章节的所有生产级最佳实践，包含完整的目录结构、配置文件、Dockerfile 与 docker-compose.yml。

### 5.1 项目目录结构


```plaintext
compose-fullstack-demo/
├── docker-compose.yml  # 核心Compose配置文件
├── .env                # 环境变量配置文件，避免硬编码
├── .dockerignore       # 构建上下文忽略文件
├── nginx/
│   ├── Dockerfile      # Nginx镜像构建文件
│   ├── nginx.conf      # 核心配置文件
│   └── html/           # 前端静态资源目录
├── backend/
│   ├── Dockerfile      # SpringBoot后端镜像构建文件（多阶段构建）
│   ├── pom.xml         # Maven配置文件
│   └── src/            # SpringBoot源码目录
├── mysql/
│   ├── my.cnf          # MySQL自定义配置文件
│   └── init/           # 数据库初始化SQL脚本目录
└── redis/
    └── redis.conf      # Redis自定义配置文件
```

### 5.2 核心配置文件

#### 5.2.1 .env 环境变量文件（避免硬编码敏感信息）

```bash
# 项目全局配置
COMPOSE_PROJECT_NAME=fullstack_app
TZ=Asia/Shanghai

# MySQL配置
MYSQL_VERSION=8.0
MYSQL_ROOT_PASSWORD=Prod@123456
MYSQL_DATABASE=app_db
MYSQL_USER=app_user
MYSQL_PASSWORD=App@123456
MYSQL_PORT=3306

# Redis配置
REDIS_VERSION=7.2-alpine
REDIS_PASSWORD=Redis@123456
REDIS_PORT=6379

# 后端服务配置
BACKEND_PORT=8080
SPRING_PROFILES_ACTIVE=prod

# Nginx配置
NGINX_VERSION=1.27.0-alpine
NGINX_HTTP_PORT=80
NGINX_HTTPS_PORT=443
```

#### 5.2.2 docker-compose.yml 核心编排文件

```yaml
services:
  # 1. Nginx反向代理服务：前端静态资源 + 后端接口反向代理
  nginx:
    build: ./nginx
    image: ${COMPOSE_PROJECT_NAME}/nginx:${NGINX_VERSION}
    ports:
      - "0.0.0.0:${NGINX_HTTP_PORT}:80"
      - "0.0.0.0:${NGINX_HTTPS_PORT}:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/html:/usr/share/nginx/html:ro
      - nginx_logs:/var/log/nginx
    networks:
      - frontend_net
      - backend_net
    depends_on:
      backend:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "nginx", "-t"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 5s

  # 2. SpringBoot后端业务服务
  backend:
    build: ./backend
    image: ${COMPOSE_PROJECT_NAME}/backend:latest
    environment:
      SPRING_PROFILES_ACTIVE: ${SPRING_PROFILES_ACTIVE}
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/${MYSQL_DATABASE}?useSSL=false&serverTimezone=Asia/Shanghai&allowPublicKeyRetrieval=true
      SPRING_DATASOURCE_USERNAME: ${MYSQL_USER}
      SPRING_DATASOURCE_PASSWORD: ${MYSQL_PASSWORD}
      SPRING_DATA_REDIS_HOST: redis
      SPRING_DATA_REDIS_PORT: ${REDIS_PORT}
      SPRING_DATA_REDIS_PASSWORD: ${REDIS_PASSWORD}
      TZ: ${TZ}
    networks:
      - backend_net
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:${BACKEND_PORT}/actuator/health"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 20s

  # 3. MySQL数据库服务
  mysql:
    image: mysql:${MYSQL_VERSION}
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      TZ: ${TZ}
    volumes:
      - mysql_data:/var/lib/mysql
      - ./mysql/my.cnf:/etc/mysql/conf.d/my.cnf:ro
      - ./mysql/init:/docker-entrypoint-initdb.d:ro
    networks:
      - backend_net
    # 生产环境严禁对外暴露端口，仅内部网络访问
    # ports:
    #   - "127.0.0.1:${MYSQL_PORT}:3306"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 20s

  # 4. Redis缓存服务
  redis:
    image: redis:${REDIS_VERSION}
    environment:
      TZ: ${TZ}
    command: redis-server /etc/redis/redis.conf --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
      - ./redis/redis.conf:/etc/redis/redis.conf:ro
    networks:
      - backend_net
    # 生产环境严禁对外暴露端口
    # ports:
    #   - "127.0.0.1:${REDIS_PORT}:6379"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3
      start_period: 5s

# 网络拓扑定义：前后端网络隔离
networks:
  frontend_net:
    driver: bridge
    labels:
      project: ${COMPOSE_PROJECT_NAME}
      env: production
  backend_net:
    driver: bridge
    labels:
      project: ${COMPOSE_PROJECT_NAME}
      env: production

# 全局数据卷定义：持久化核心业务数据
volumes:
  mysql_data:
    labels:
      project: ${COMPOSE_PROJECT_NAME}
      data: mysql
  redis_data:
    labels:
      project: ${COMPOSE_PROJECT_NAME}
      data: redis
  nginx_logs:
    labels:
      project: ${COMPOSE_PROJECT_NAME}
      data: nginx_logs
```

#### 5.2.3 后端服务 Dockerfile（多阶段构建，对应第七章最佳实践）

```dockerfile
# 构建阶段：Maven编译环境
FROM eclipse-temurin:17-jdk-alpine AS builder
WORKDIR /app
# 复制pom.xml，优先下载依赖，最大化缓存复用
COPY pom.xml .
RUN mvn dependency:go-offline -B
# 复制源码并编译打包
COPY src ./src
RUN mvn clean package -DskipTests -Dmaven.test.skip=true

# 运行阶段：精简JRE环境
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
# 安装健康检查依赖curl
RUN apk add --no-cache curl
# 仅复制构建阶段的编译产物
COPY --from=builder /app/target/*.jar app.jar
# 降权运行：创建非root用户
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
# 暴露服务端口
EXPOSE 8080
# 健康检查端点
HEALTHCHECK --interval=5s --timeout=3s --retries=5 --start-period=20s CMD curl -f http://localhost:8080/actuator/health || exit 1
# 启动命令
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 5.3 核心设计说明与生产规范

1. **网络隔离设计**：划分`frontend_net`与`backend_net`两个独立网络，Nginx 同时接入两个网络，实现流量转发；MySQL、Redis、后端服务仅接入`backend_net`，与前端网络完全隔离，数据库、缓存不对外暴露任何端口，严格遵循最小权限原则；
2. **依赖与就绪控制**：所有服务均配置健康检查，通过`condition: service_healthy`严格控制启动顺序：MySQL/Redis 就绪 → 后端服务就绪 → Nginx 启动，彻底避免启动失败；
3. **数据持久化**：所有业务数据、日志均使用项目级命名卷持久化，容器销毁、重建不影响数据；配置文件均使用`ro`只读模式挂载，避免误修改；
4. **环境变量管理**：所有配置、敏感信息均通过`.env`文件管理，无硬编码，配置与代码完全分离，适配多环境部署；
5. **高可用保障**：所有服务均配置`restart: unless-stopped`重启策略，容器异常退出时自动重启，保障服务可用性；
6. **安全加固**：后端服务使用非 root 用户降权运行，数据库、缓存不对外暴露端口，所有配置文件只读挂载，缩小攻击面。

### 5.4 一键部署与验证

1. **环境准备**：确保已安装 Docker Engine 与 Compose V2 插件，执行`docker compose version`验证环境正常；
2. **一键启动**：在项目根目录执行`docker compose up -d`，Compose 会自动完成镜像构建、网络创建、卷创建、服务启动全流程；
3. **状态验证**：执行`docker compose ps`查看所有服务状态，所有服务状态均为`Up`且健康状态为`healthy`，表示部署成功；
4. **日志排查**：若服务启动异常，执行`docker compose logs -f 服务名`查看日志，定位问题；
5. **停止与销毁**：执行`docker compose down`停止并销毁所有容器、网络，数据卷会保留，不会丢失业务数据。

## 本章小结

本章完整讲解了 Docker Compose 多容器编排的核心知识体系，覆盖核心定位、配置语法、依赖管理、高频命令与全栈实战演练，核心知识点如下：

1. 理解 Docker Compose 的核心定位、解决的痛点，掌握项目与服务两大核心概念；
2. 熟练掌握`docker-compose.yml`的四大核心模块（services、networks、volumes）的语法规则、底层逻辑与生产级配置规范；
3. 深度理解`depends_on`的核心局限性，掌握`healthcheck`健康检查的语法与最佳实践，实现服务依赖与就绪状态的完整管控；
4. 熟练掌握 Compose 全生命周期的高频命令，明确生产环境的使用规范与风险边界；
5. 能够基于 Compose 实现完整的多容器微服务应用的一键部署，遵循网络隔离、数据持久化、安全加固、高可用的生产级规范。

本章内容是单主机多容器应用部署的标准解决方案，是本地开发、测试环境的核心工具，也是向 Kubernetes 集群编排过渡的重要基础，需完全掌握所有知识点与最佳实践后，再进入后续底层原理、企业级实战章节的学习。