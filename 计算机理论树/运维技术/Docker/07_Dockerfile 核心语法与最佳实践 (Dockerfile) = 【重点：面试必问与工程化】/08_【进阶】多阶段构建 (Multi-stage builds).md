
多阶段构建是 Docker 17.05 + 版本支持的核心特性，是解决镜像体积过大、构建环境与运行环境分离、减少攻击面的核心方案，是面试高频考点。

### 8.1 核心设计目标与解决的痛点

传统单阶段构建的致命缺陷：对于编译型语言（Go、Java、C++ 等）、前端项目，构建过程需要完整的编译环境、依赖工具、源码，而运行时仅需要最终的编译产物。单阶段构建会将所有编译环境、源码、中间产物都打包到最终镜像中，导致：

1. 镜像体积巨大，拉取、推送速度极慢，占用大量存储与带宽资源；
2. 镜像包含大量无用的编译工具、依赖包，攻击面极广，存在大量安全漏洞风险；
3. 构建与运行环境耦合，不符合最小镜像的生产安全规范。

多阶段构建的核心设计目标：**将构建环境与运行环境完全分离**，通过多个 FROM 指令定义多个构建阶段，前序阶段完成编译、打包、依赖安装等构建操作，最终阶段仅复制前序阶段的最终运行产物，基于极简的运行时基础镜像构建最终镜像，所有编译环境、中间产物、源码全部丢弃，实现极致的镜像体积优化与安全加固。

### 8.2 核心语法规则

1. Dockerfile 中每一个 FROM 指令定义一个独立的构建阶段，每个阶段可独立命名，拥有自己的基础镜像与构建指令，阶段之间互相隔离；
2. 通过`FROM ... AS <阶段名>`为构建阶段命名，后续阶段可通过`COPY --from=<阶段名>`复制该阶段的文件；
3. Dockerfile 中最后一个 FROM 阶段定义的镜像，就是最终生成的镜像，之前的所有阶段仅用于构建，不会包含在最终镜像中；
4. 每个阶段的构建缓存独立，仅当阶段内的指令或依赖变更时，才会重新构建该阶段。

### 8.3 典型场景实战

#### 场景 1：Go 语言项目多阶段构建

Go 语言编译后生成无依赖的静态二进制文件，配合 distroless 镜像，可实现极致的镜像体积优化。

```dockerfile
# 构建阶段：命名为builder，使用完整的Go编译环境
FROM golang:1.22-alpine AS builder
WORKDIR /app
# 复制依赖管理文件，优先安装依赖，最大化缓存复用
COPY go.mod go.sum ./
RUN go mod download
# 复制源码
COPY . .
# 静态编译，禁用CGO，生成无依赖的二进制文件
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app main.go

# 最终运行阶段：使用极简的distroless镜像，无Shell、无包管理
FROM gcr.io/distroless/static-debian12
WORKDIR /app
# 仅从builder阶段复制编译好的二进制文件，其他所有内容全部丢弃
COPY --from=builder /app/app .
# 声明端口
EXPOSE 8080
# 启动程序
ENTRYPOINT ["/app/app"]
```

优化效果：传统单阶段构建镜像体积 > 300MB，多阶段构建最终镜像体积 < 15MB，缩小 95% 以上，无任何无用依赖，攻击面极小。

#### 场景 2：Java SpringBoot 项目多阶段构建

Java 项目需要 JDK 完成编译打包，运行时仅需要 JRE 环境，多阶段构建可分离编译与运行环境。

```dockerfile
# 构建阶段：使用完整的JDK环境，命名为builder
FROM eclipse-temurin:17-jdk-alpine AS builder
WORKDIR /app
# 复制Maven配置与pom.xml，优先下载依赖，缓存复用
COPY mvnw pom.xml ./
COPY .mvn ./.mvn
RUN ./mvnw dependency:go-offline
# 复制源码
COPY src ./src
# 编译打包，跳过单元测试
RUN ./mvnw package -DskipTests

# 最终运行阶段：使用精简的JRE环境，仅包含运行时依赖
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
# 仅从builder阶段复制打包好的jar包
COPY --from=builder /app/target/*.jar app.jar
# 声明端口
EXPOSE 8080
# 启动命令
ENTRYPOINT ["java", "-jar", "app.jar"]
```

#### 场景 3：前端项目多阶段构建（React/Vue）

前端项目需要 Node.js 环境完成编译打包，运行时仅需要编译后的静态文件，配合 Nginx 镜像实现部署。

```dockerfile
# 构建阶段：使用Node.js环境，完成前端项目编译
FROM node:20-alpine AS builder
WORKDIR /app
# 复制package.json，优先安装依赖，缓存复用
COPY package.json package-lock.json ./
RUN npm ci
# 复制源码
COPY . .
# 编译打包，生成静态文件
RUN npm run build

# 最终运行阶段：使用极简的Nginx alpine镜像，仅提供静态文件服务
FROM nginx:1.27.0-alpine
# 复制编译后的静态文件到Nginx的静态资源目录
COPY --from=builder /app/dist /usr/share/nginx/html
# 复制自定义Nginx配置
COPY nginx.conf /etc/nginx/nginx.conf
# 声明端口
EXPOSE 80
# 启动Nginx
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
```

### 8.4 生产级最佳实践

1. 最终运行阶段必须使用最小化的基础镜像，优先选择 alpine、distroless，避免使用完整的发行版镜像；
2. 构建阶段与运行阶段分离，最终镜像中仅包含运行必需的产物，严禁包含源码、编译工具、中间产物；
3. 合理拆分构建步骤，最大化构建缓存复用，将变更频率低的步骤放在前面；
4. 可在多阶段构建中增加测试阶段，完成单元测试、代码扫描、安全检测，测试不通过则构建失败，实现安全左移；
5. 配合 buildx 实现多架构镜像构建，一次构建生成支持 amd64/arm64 等多架构的镜像。

## 生产级 Dockerfile 核心最佳实践汇总

1. **可复现性保障**：所有基础镜像、软件包必须指定固定版本 / 摘要，严禁使用 latest 标签，保证镜像构建的可复现性；
2. **镜像体积最小化**：使用多阶段构建分离构建与运行环境，选择最小化基础镜像，单条 RUN 指令完成安装与清理，避免无用依赖与缓存残留；
3. **构建缓存最大化**：合理排序 Dockerfile 指令，变更频率低的指令在前，变更频率高的在后；先复制依赖配置文件，安装依赖，再复制源码，最大化缓存复用；
4. **安全加固**：
    
    - 严禁使用 root 用户运行业务进程，通过 USER 指令降权，创建非 root 用户运行应用；
    - 生产环境使用只读容器根文件系统，仅开放必要的 tmpfs 挂载点用于临时写入；
    - 禁用不必要的 Linux Capabilities，严禁使用 --privileged 特权模式；
    - 严禁在 Dockerfile 中存储任何敏感信息，敏感信息通过构建密钥、运行时 Secret 实现；
    
5. **可维护性规范**：
    
    - 必须配套.dockerignore 文件，最小化构建上下文；
    - 必须通过 LABEL 声明标准化元数据，实现镜像可追溯；
    - 必须通过 EXPOSE、VOLUME 声明端口与数据路径，提供标准化文档；
    
6. **进程管理规范**：
    
    - CMD/ENTRYPOINT 必须使用 Exec 格式，严禁使用 Shell 格式；
    - 容器 1 号进程必须是业务主进程，保证信号传递正常，实现优雅停止；
    - 业务进程必须前台运行，严禁后台守护进程模式启动。
## 面试高频核心考点汇总

1. Dockerfile 中 COPY 与 ADD 的核心区别是什么？生产环境优先使用哪个？为什么？
2. CMD 与 ENTRYPOINT 的核心区别是什么？生产环境的最佳配合方式是什么？
3. 什么是多阶段构建？解决了什么问题？核心优势是什么？
4. RUN、CMD、ENTRYPOINT 的区别是什么？分别在什么阶段执行？
5. Dockerfile 的每一条指令都会生成镜像分层吗？哪些指令会生成物理分层？哪些仅修改元数据？
6. EXPOSE 指令会自动创建端口映射吗？它的核心作用是什么？
7. VOLUME 指令的核心陷阱是什么？为什么声明 VOLUME 后对该路径的修改会失效？
8. docker build . 中的。代表什么？构建上下文的本质是什么？.dockerignore 的作用是什么？
9. ENV 与 ARG 的核心区别是什么？各自的生效周期是什么？
10. 为什么 Shell 格式的 CMD/ENTRYPOINT 会导致容器无法优雅停止？底层原理是什么？

## 本章小结

本章完整覆盖了 Dockerfile 的全量核心语法、底层执行逻辑、镜像分层关联、生产级最佳实践与面试高频考点，核心知识点如下：

1. 掌握 Dockerfile 的本质与镜像分层的关联关系，理解每一条指令的执行逻辑与对镜像分层的影响；
2. 掌握 FROM、LABEL、ENV、ARG、RUN、COPY、ADD、CMD、ENTRYPOINT、EXPOSE、VOLUME 等核心指令的语法、底层特性、生产规范与避坑要点；
3. 深度理解 CMD 与 ENTRYPOINT 的核心区别，掌握生产环境的最佳配合方式，解决容器优雅停止的核心问题；
4. 理解构建上下文的本质，掌握.dockerignore 的编写规范与构建优化方法；
5. 深度掌握多阶段构建的核心原理、语法、典型场景实战，实现镜像体积的极致优化与安全加固；
6. 建立生产级 Dockerfile 的标准化编写规范，掌握面试高频考点的核心答案。

本章内容是 Docker 镜像构建的核心，是企业级容器化工程化、DevOps 流水线的核心基础，需完全掌握所有知识点与最佳实践后，再进入后续 Docker Compose 编排、底层原理等章节的学习。