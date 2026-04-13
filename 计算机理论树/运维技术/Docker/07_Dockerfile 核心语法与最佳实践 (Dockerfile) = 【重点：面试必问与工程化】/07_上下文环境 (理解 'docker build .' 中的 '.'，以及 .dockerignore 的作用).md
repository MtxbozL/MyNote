### 7.1 构建上下文的核心本质

`docker build`命令末尾的路径（最常用的是`.`，当前目录），就是 Docker 构建的**上下文根路径**，是新手最容易误解的核心概念。

- 底层执行流程：
    
    1. 执行`docker build`命令时，Docker Client 会将上下文根路径下的**所有文件与目录**（包括子目录）完整打包，通过 REST API 发送给 Docker Daemon；
    2. Docker Daemon 只能访问该打包文件内的内容，所有 COPY/ADD 指令的源路径，都必须是上下文内的相对路径，无法访问上下文外的任何文件（如`../`开头的路径），这就是`COPY ../xxx`会报错的核心原因；
    3. 构建完成后，Docker Daemon 会删除该临时打包文件，不会保留。
    
- 核心误区纠正：`.`不是 Dockerfile 所在的路径，而是构建上下文的根路径；Dockerfile 默认从上下文根路径查找，可通过`-f`参数指定 Dockerfile 的路径，与上下文路径分离。示例：

    ```bash
    # 上下文是当前目录，Dockerfile位于./build/Dockerfile
    docker build -f ./build/Dockerfile .
    ```
    
- 构建上下文的核心影响：
    
    1. 上下文包含大量无用文件时，会导致构建启动缓慢，甚至占用大量网络带宽；
    2. 上下文中的无关文件变更，会导致构建缓存失效，无法复用之前的构建分层；
    3. 敏感文件被包含在上下文中，存在泄露风险。
### 7.2 .dockerignore 文件

用于定义构建上下文的排除规则，与`.gitignore`语法一致，Docker Client 打包上下文时，会自动排除匹配规则的文件与目录，是优化构建速度、缩小上下文、提升安全性的核心工具。

- 核心作用：
    
    1. 排除无用文件，减少上下文打包体积，大幅提升构建速度；
    2. 避免敏感文件被打包到上下文中，防止敏感信息泄露；
    3. 排除与构建无关的文件变更，避免构建缓存无故失效。
    
- 生产环境标准化示例：

    ```plaintext
    # 版本控制目录
    .git
    .gitignore
    .github
    
    # 依赖目录
    node_modules
    vendor
    venv
    
    # 临时文件与日志
    *.log
    tmp/
    temp/
    
    # 构建产物
    dist/
    build/
    *.tar.gz
    *.zip
    
    # 敏感文件
    .env
    .env.*
    *.pem
    *.key
    id_rsa
    
    # 本地配置文件
    docker-compose.override.yml
    .dockerignore
    Dockerfile
    ```
    
- 生产级最佳实践：
    
    1. **所有 Dockerfile 必须配套对应的.dockerignore 文件**，最小化构建上下文，这是构建优化的第一步；
    2. 遵循白名单原则：先排除所有文件`*`，再通过`!`规则仅包含构建必需的文件，最大化缩小上下文；
    3. 严禁将敏感文件包含在上下文中，必须通过.dockerignore 排除。
    