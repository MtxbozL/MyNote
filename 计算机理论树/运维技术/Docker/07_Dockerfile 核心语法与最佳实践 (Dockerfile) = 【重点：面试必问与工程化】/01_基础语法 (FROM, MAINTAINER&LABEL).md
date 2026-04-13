## 本章导读

Dockerfile 是 Docker 镜像构建的标准化声明式配置文件，是实现镜像可复现、可追溯、自动化构建的核心载体，完全承接第三章的镜像分层存储原理，是云原生「基础设施即代码（IaC）」理念的核心落地形式，也是企业级容器化工程化、DevOps 流水线的核心环节，同时是 Docker 相关面试的**最高频核心考点**。

本章将完整覆盖 Dockerfile 全量核心语法、底层执行逻辑、生产级最佳实践、镜像体积优化方案，严格遵循 OCI 镜像规范，内容兼顾学术严谨性与工程实用性，无遗漏覆盖目录内所有知识点，同时补充面试必问的核心原理与避坑规范。

### 前置核心原理：Dockerfile 与镜像分层的本质关联

Docker Daemon 按顺序执行 Dockerfile 中的每一条指令，**每一条可产生文件系统变更的指令（FROM、RUN、COPY、ADD 等）都会生成一个独立的只读镜像分层**，完全匹配第三章的 Overlay2 分层存储架构；无文件系统变更的元数据指令（LABEL、EXPOSE、ENV、CMD 等）仅修改镜像元数据，不会生成新的物理分层，但会影响镜像 ID 与构建缓存。Dockerfile 的优化核心，本质是对镜像分层的合理设计，兼顾构建缓存复用、镜像体积、可维护性与安全性。

## 01 基础语法 (FROM, MAINTAINER/LABEL)

### 1.1 FROM 指令

FROM 是 Dockerfile 的**第一条有效指令**，也是唯一不可省略的指令，用于指定构建的基础镜像，所有后续指令都基于该基础镜像的文件系统执行。

- 标准语法：

    ```dockerfile
    FROM [--platform=<平台>] <基础镜像>[:<标签>|@<摘要>] [AS <阶段名>]
    ```
    
- 核心参数详解：
    
    1. `--platform`：指定基础镜像的平台架构，如`linux/amd64`、`linux/arm64`，用于多架构镜像构建，解决跨平台构建兼容性问题；
    2. 镜像引用规范：生产环境**必须通过摘要 @<SHA256>或固定版本标签指定镜像**，严禁使用`latest`标签，避免基础镜像版本漂移导致的构建不可复现、兼容性故障；
    3. `AS <阶段名>`：为构建阶段命名，用于多阶段构建的产物引用，单阶段构建可省略。
    
- 基础镜像选型生产规范：
    
    表格
    
    |镜像类型|核心特点|适用场景|生产风险|
    |---|---|---|---|
    |官方发行版镜像（ubuntu/debian/centos）|完整的 Linux 发行版环境，包管理完善，兼容性强|复杂依赖的应用，需要完整系统工具的场景|镜像体积大，攻击面广，安全漏洞多|
    |slim 变体镜像|基于发行版裁剪的精简版本，仅保留核心运行环境|大部分服务端应用，平衡兼容性与体积|部分系统工具缺失，需适配依赖|
    |alpine 镜像|基于 musl libc 与 busybox 的极简镜像，体积通常 < 10MB|运行时依赖少的应用，如 Go、C++ 编译产物|musl libc 与 glibc 存在兼容性差异，部分应用无法正常运行|
    |distroless 镜像|Google 出品的无发行版镜像，仅包含应用运行所需的最小依赖，无 Shell、无包管理|编译型语言生产部署，极致安全与体积优化|无调试工具，排障难度大，仅适用于成熟稳定的生产应用|
    
- 特殊合法用法：`ARG`指令可位于 FROM 之前，用于定义 FROM 中使用的变量，是唯一可位于 FROM 之前的指令，示例：
    
    dockerfile
    
    ```
    ARG NGINX_VERSION=1.27.0
    FROM nginx:${NGINX_VERSION}-alpine
    ```
    
    注意：FROM 之前的 ARG 仅在 FROM 指令中生效，构建阶段内的 ARG 需要重新定义。

### 1.2 MAINTAINER 与 LABEL 指令

- `MAINTAINER`：**已被 Docker 官方完全废弃**，仅保留兼容性支持，原用于声明镜像作者信息，生产环境严禁使用，完全由 LABEL 指令替代。
- `LABEL`：用于为镜像添加键值对形式的元数据，可用于声明作者、版本、业务归属、许可证、合规信息等，支持多行定义，不会生成新的镜像分层，仅修改镜像元数据。
    
    - 标准语法：
        
        dockerfile
        
        ```
        LABEL <key>=<value> <key2>=<value2> ...
        ```
        
    - 生产规范：
        
        1. 必须声明核心元数据，实现镜像的可追溯性，遵循 OCI 镜像规范的标准标签前缀`org.opencontainers.image`，提升元数据的标准化与兼容性；
        2. 多行合并定义，避免多条 LABEL 指令，提升可读性。
            
            标准化示例：
        
        dockerfile
        
        ```
        LABEL org.opencontainers.image.authors="dev@example.com" \
              org.opencontainers.image.version="1.0.0" \
              org.opencontainers.image.description="生产环境Nginx服务镜像" \
              org.opencontainers.image.licenses="MIT" \
              business.owner="支付业务部" \
              env="production"
        ```
        
    
