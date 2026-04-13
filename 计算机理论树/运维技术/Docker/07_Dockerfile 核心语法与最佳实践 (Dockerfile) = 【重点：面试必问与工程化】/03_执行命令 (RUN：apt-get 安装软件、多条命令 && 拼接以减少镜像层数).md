
RUN 指令用于在镜像构建阶段执行指定的命令，命令执行结果会写入镜像分层，是生成镜像变更的核心指令，**每一条 RUN 指令都会生成一个独立的镜像分层**。

### 3.1 两种执行格式与底层差异

1. **Exec 格式（官方推荐，生产优先）**

    ```dockerfile
    RUN ["executable", "param1", "param2"]
    ```
    
    底层逻辑：直接执行指定的可执行文件，不会启动`/bin/sh` Shell 环境，不触发 Shell 变量解析、管道、重定向等特性；优势是避免 Shell 进程注入，无 Shell 依赖，适配无 Shell 的 distroless 镜像。
    
2. **Shell 格式（默认）**

    ```dockerfile
    RUN <command>
    ```
    
    底层逻辑：默认以`/bin/sh -c <command>`的方式执行命令，支持所有 Shell 特性，如变量解析、管道、重定向、逻辑运算符；适用于需要 Shell 特性的复杂命令执行，如包管理安装、多命令拼接。

### 3.2 生产级最佳实践（面试必问）

1. **相关命令合并，减少镜像分层**
    
    将逻辑关联的多条命令通过`&&`拼接为单条 RUN 指令，避免生成大量无用分层；通过`\`实现换行，提升可读性。同时必须在同一条指令中完成包更新、安装、缓存清理，避免缓存分层导致的安装包版本过期与镜像体积膨胀。
    
    标准化示例：

    ```dockerfile
    RUN apt-get update && \
        apt-get install -y --no-install-recommends nginx=1.27.0-* && \
        apt-get clean && \
        rm -rf /var/lib/apt/lists/*
    ```
    
2. **--no-install-recommends 强制规范**
    
    Debian/Ubuntu 系列包管理必须添加该参数，仅安装依赖的必需包，不安装推荐的可选包，大幅减少不必要的依赖安装，缩小镜像体积。
    
3. **安装后强制清理缓存**
    
    包管理安装完成后，必须在同一条 RUN 指令中执行清理命令，删除缓存文件、临时文件，避免缓存保留在镜像分层中。不同发行版清理命令：
    
    - Debian/Ubuntu：`rm -rf /var/lib/apt/lists/*`
    - RHEL/CentOS/Fedora：`dnf clean all`
    - Alpine：`rm -rf /var/cache/apk/*`
    
4. **固定软件包版本**
    
    安装软件时必须指定固定版本，如`nginx=1.27.0-*`，避免构建时安装的版本不一致，导致镜像不可复现。
    
5. **构建缓存优化**
    
    将变更频率低的命令放在 Dockerfile 前部，变更频率高的命令放在后部，最大化缓存复用。例如：先安装系统依赖，再复制依赖配置文件，再安装应用依赖，最后复制应用源码。
    
