
全局块为`nginx.conf`的最顶层配置块，无父块包裹，配置指令作用于**整个 Nginx 进程生命周期**，核心用于定义进程运行的基础环境参数，部分指令仅在 Nginx 启动时加载，修改后需重启服务方可生效，不可通过`reload`热重载。

### 2.2.1 核心配置指令

1. **user**
    
    - 语法：`user 用户名 [用户组];`
    - 作用：指定 Worker 进程的运行系统用户与用户组，决定 Nginx 对服务器文件、目录的访问权限，默认值通常为`nobody`或`nginx`；
    - 示例：`user www www;`（Worker 进程以 www 用户、www 用户组运行）；
    - 注意：需确保指定用户对静态资源目录、日志目录拥有对应读写权限，否则会出现 403 错误或日志写入失败。
    
2. **worker_processes**
    
    - 语法：`worker_processes 数字 | auto;`
    - 作用：指定 Nginx 启动的 Worker 工作进程数，是 Nginx 多核 CPU 利用的核心参数；
    - 最佳实践：设置为`auto`（Nginx 自动检测 CPU 物理核心数）或等于 / 略大于 CPU 物理核心数，避免进程数过多导致 CPU 调度开销激增；
    - 示例：`worker_processes auto;`（推荐配置）。
    
3. **error_log**
    
    - 语法：`error_log 日志路径 日志级别;`
    - 作用：定义 Nginx 全局错误日志的存储路径与日志级别，记录 Nginx 启动、运行、关闭过程中的错误信息；
    - 日志级别（从低到高）：`debug > info > notice > warn > error > crit > alert > emerg`，级别越高记录的日志越少，生产环境推荐`warn`或`error`；
    - 示例：`error_log /var/log/nginx/error.log warn;`；
    - 注意：`debug`级别仅在 Nginx 编译时启用`--with-debug`参数后生效，且会产生大量日志，禁止在生产环境使用。
    
4. **pid**
    
    - 语法：`pid 文件路径;`
    - 作用：指定 Nginx Master 主进程的 PID 文件存储路径，PID 文件记录 Master 进程的进程号，用于进程管理与信号通信；
    - 示例：`pid /var/run/nginx.pid;`；
    - 注意：需确保 Nginx 运行用户对 PID 文件所在目录拥有写入权限。
    
5. **worker_rlimit_nofile**
    
    - 语法：`worker_rlimit_nofile 数字;`
    - 作用：提升单个 Worker 进程能打开的最大文件描述符数量，解决 Linux 系统默认文件描述符限制导致的高并发连接失败问题；
    - 最佳实践：设置为较大值，如`65535`，需配合修改 Linux 系统内核参数`ulimit -n`使用；
    - 示例：`worker_rlimit_nofile 65535;`。

### 2.2.2 配置注意事项

- 全局块指令**不可在子块中重复配置**，如`worker_processes`不可在 events 块或 http 块中定义；
- 涉及系统资源限制的指令（如`worker_rlimit_nofile`）需与 Linux 系统内核参数配合，否则配置不生效；
- 日志路径、PID 文件路径需提前创建目录，并赋予 Nginx 运行用户对应权限。
