
Events 块是`main`全局块的直接子块，核心用于配置 Nginx**网络连接处理的底层参数**，作用于所有 Worker 进程，指令主要围绕事件驱动模型、并发连接数优化，是 Nginx 高并发能力的基础配置，修改后需通过`reload`热重载生效。

### 2.3.1 核心配置指令

1. **worker_connections**
    
    - 语法：`worker_connections 数字;`
    - 作用：指定单个 Worker 进程最大可同时处理的并发连接数，是 Nginx 并发能力的核心指标；
    - 取值限制：受`worker_rlimit_nofile`与 Linux 系统文件描述符限制，最大值不可超过`worker_rlimit_nofile`配置值；
    - 示例：`worker_connections 10240;`（单个 Worker 进程支持 10240 个并发连接）；
    - 总并发数估算：Nginx 最大理论并发数 = `worker_processes` × `worker_connections` / 2（因 HTTP 连接为双向连接）。
    
2. **use**
    
    - 语法：`use 事件模型;`
    - 作用：指定 Nginx 使用的 IO 多路复用事件驱动模型，不同操作系统支持的模型不同；
    - 主流模型：Linux 系统推荐`epoll`（高性能，支持海量并发），FreeBSD 系统使用`kqueue`，Windows 系统使用`iocp`，低版本 Linux 使用`poll/select`；
    - 示例：`use epoll;`（生产环境推荐配置，Nginx 高版本默认自动适配最优模型）。
    
3. **multi_accept**
    
    - 语法：`multi_accept on | off;`
    - 作用：设置 Worker 进程是否一次性接收所有可用的新连接，开启后可提升高并发下的连接处理效率；
    - 示例：`multi_accept on;`（推荐开启）。
    

### 2.3.2 配置优化原则

- `worker_connections`需与`worker_processes`配合，避免单进程连接数过大或进程数过多导致的资源浪费；
- 无需手动指定`use`指令，Nginx 高版本会根据操作系统自动选择最优事件模型，手动配置可能导致适配错误；
- 配置值需结合服务器硬件配置（如内存、CPU）调整，并非越大越好，避免超出系统资源承载能力。
