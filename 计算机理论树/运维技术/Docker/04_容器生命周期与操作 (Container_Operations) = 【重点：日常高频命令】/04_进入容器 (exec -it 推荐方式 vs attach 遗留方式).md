
进入容器是指进入容器的 Namespace 隔离环境，对容器内的文件系统、进程、配置进行操作与调试，核心分为官方推荐的`docker exec`方式与遗留的`docker attach`方式，需严格区分二者的底层差异与适用边界。

### 4.1 docker exec：官方推荐方式

`docker exec`是生产环境进入容器、执行容器内命令的唯一推荐方式，可在运行中的容器内创建新的独立进程，加入容器的所有 Namespace 隔离环境，实现对容器的安全访问。

#### 4.1.1 核心语法与参数

```bash
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
```

- 核心必用参数：`-it`，与 run 命令的`-it`逻辑一致，为 exec 创建的进程分配伪终端并保持标准输入开启，实现交互式 Shell 访问；
- 典型用法：
    
    1. 交互式进入容器终端：`docker exec -it nginx /bin/bash`（Alpine 镜像需使用`/bin/sh`）；
    2. 执行一次性命令，无需进入终端：`docker exec nginx cat /etc/nginx/nginx.conf`；
    
- 补充参数：`-u/--user`，指定执行命令的用户，生产环境可用于降权访问，避免直接使用 root 用户操作。

#### 4.1.2 底层原理与核心优势

- 底层逻辑：在运行中的容器的 PID、Network、MNT 等所有 Namespace 中，创建一个全新的独立进程，该进程与容器的 1 号进程共享隔离环境，可访问容器内的所有资源；
- 核心优势：
    
    1. 完全不影响容器的 1 号进程，退出 exec 终端时，不会向 1 号进程发送任何信号，不会导致容器停止，彻底规避 attach 方式的致命缺陷；
    2. 支持多终端同时 exec 进入同一容器，操作互不干扰，无同步显示问题；
    3. 支持执行一次性命令与交互式访问，适配运维脚本、人工调试等全场景；
    
- 限制条件：仅可对**运行中**的容器执行 exec 操作，已停止的容器无法使用。

### 4.2 docker attach：遗留方式

`docker attach`是 Docker 早期的容器访问方式，核心作用是将当前终端的标准输入、输出、错误流，附加到容器 1 号进程的标准流上，直接连接到容器启动时的主进程终端。

#### 4.2.1 核心语法

```bash
docker attach [OPTIONS] CONTAINER
```

#### 4.2.2 核心缺陷与禁用边界

`docker attach`存在致命的设计缺陷，**生产环境严禁使用**，仅可用于调试容器 1 号进程的启动输出，核心缺陷如下：

1. **容器意外停止风险**：退出 attach 终端时，若按下`Ctrl+C`，会直接向容器 1 号进程发送 SIGINT 信号，导致 1 号进程退出，容器立即停止，这是最高频的致命坑点；
2. 多终端干扰：多个终端同时 attach 到同一容器时，所有终端的操作会同步显示，输入输出互相干扰，无法正常运维；
3. 交互能力受限：若容器 1 号进程为非交互式后台进程（如 Nginx、MySQL），attach 后无法输入任何命令，无交互效果；
4. 不支持一次性命令执行，仅能附加到主进程的标准流，灵活性极差。

### 4.3 底层原生方式：nsenter

`nsenter`是 Linux util-linux 包提供的原生工具，可直接进入进程的 Namespace 隔离环境，是`docker exec`的底层实现基础，适用于容器 1 号进程异常、docker exec 无法使用的极端排障场景。

#### 典型用法

1. 获取容器 1 号进程在宿主机上的 PID：`PID=$(docker inspect --format '{{.State.Pid}}' 容器名)`
2. 进入容器的所有 Namespace：`nsenter -t $PID -m -u -i -n -p`

- 参数说明：`-m`挂载 Namespace、`-u`UTS Namespace、`-i`IPC Namespace、`-n`Network Namespace、`-p`PID Namespace；
- 核心优势：完全绕过 Docker Daemon，直接通过内核机制进入容器隔离环境，即使 Docker Daemon 异常，也可正常访问容器，是生产环境极端故障的终极排障手段。
