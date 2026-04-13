
CMD 与 ENTRYPOINT 是定义容器启动时执行的命令与入口程序的核心指令，共同决定了容器的 1 号进程，容器的生命周期完全绑定该进程的运行状态，承接第四章的容器生命周期核心规则，是本章重中之重、面试 100% 高频考点。

### 5.1 CMD 指令

用于定义容器启动时**默认执行的命令与默认参数**，核心作用是为容器提供默认的启动行为，可被`docker run`命令行末尾的参数完全覆盖。

- 三种语法格式：
    
    1. **Exec 格式（官方唯一推荐）**

        ```dockerfile
        CMD ["executable", "param1", "param2"]
        ```
        
        底层逻辑：直接启动指定的可执行文件作为容器的 1 号进程，不启动 Shell 环境，支持信号传递，可正确接收 SIGTERM 信号实现优雅停止；是生产环境必须使用的格式。
    2. **ENTRYPOINT 传参格式**

        ```dockerfile
        CMD ["param1", "param2"]
        ```
        
        底层逻辑：仅为 ENTRYPOINT 指令提供默认参数，当 Dockerfile 中同时定义了 ENTRYPOINT 时，CMD 的内容会作为 ENTRYPOINT 的默认参数传递；是与 ENTRYPOINT 配合的标准格式。
    3. **Shell 格式（生产环境严禁使用）**

        ```dockerfile
        CMD command param1 param2
        ```
        
        致命缺陷：以`/bin/sh -c`的方式启动，容器的 1 号进程是`/bin/sh`，而非业务进程，Shell 进程不会转发 SIGTERM 信号给业务子进程，`docker stop`时业务进程无法收到停止信号，超过超时时间后会被 SIGKILL 强制终止，导致业务无法优雅停止，引发数据丢失、连接异常断开。
### 5.2 ENTRYPOINT 指令

用于定义容器的**入口程序**，是容器启动时的核心执行程序，不会被`docker run`命令行末尾的参数覆盖，反而会将命令行参数作为参数传递给 ENTRYPOINT 程序。

- 两种语法格式：
    
    1. **Exec 格式（官方唯一推荐）**

        ```dockerfile
        ENTRYPOINT ["executable", "param1", "param2"]
        ```
        
        底层逻辑：直接启动指定的可执行文件作为容器的 1 号进程，不启动 Shell 环境，支持信号传递；`docker run`命令行末尾的参数会追加到 ENTRYPOINT 的参数列表中，覆盖 CMD 定义的默认参数；是生产环境必须使用的格式。
    2. **Shell 格式（生产环境严禁使用）**

        ```dockerfile
        ENTRYPOINT command param1 param2
        ```
        
        致命缺陷：以`/bin/sh -c`的方式启动，1 号进程是`/bin/sh`，会忽略所有 CMD 的参数与`docker run`传递的参数，无法实现参数动态传递，同时存在信号传递失效的问题。
    

### 5.3 核心区别对比（面试必问）

|对比维度|CMD|ENTRYPOINT|
|---|---|---|
|核心作用|定义容器启动的默认命令与默认参数|定义容器的固定入口程序|
|docker run 参数覆盖规则|命令行末尾的参数会完全覆盖 CMD 的内容|命令行末尾的参数不会覆盖 ENTRYPOINT，会作为参数追加到 ENTRYPOINT 的参数列表中|
|多指令生效规则|Dockerfile 中多条 CMD 仅最后一条生效|Dockerfile 中多条 ENTRYPOINT 仅最后一条生效|
|配合逻辑|当存在 ENTRYPOINT 时，CMD 的内容作为 ENTRYPOINT 的默认参数|当存在 CMD 时，ENTRYPOINT 会接收 CMD 的默认参数，同时支持运行时参数覆盖|
|适用场景|提供可被覆盖的默认启动参数、默认命令|定义容器不可被覆盖的固定入口程序，如启动脚本、应用主程序|

### 5.4 生产环境最佳配合方式（官方标准）

**ENTRYPOINT（Exec 格式）定义固定的入口程序，CMD（Exec 格式）定义默认的启动参数**。

该模式兼顾了镜像的固定行为与灵活性：

1. 入口程序固定，不会被运行时参数覆盖，保证容器的核心启动逻辑稳定；
2. 默认参数可被`docker run`的命令行参数覆盖，支持运行时灵活调整配置；
3. 采用 Exec 格式，保证 1 号进程是业务入口程序，支持信号传递，实现优雅停止。

#### 典型生产示例 1：Nginx 镜像

```dockerfile
FROM nginx:1.27.0-alpine
# 固定入口程序
ENTRYPOINT ["nginx"]
# 默认参数，可被运行时参数覆盖
CMD ["-g", "daemon off;"]
```

- 默认启动：`docker run nginx`，执行 `nginx -g daemon off;`
- 自定义参数启动：`docker run nginx -v`，执行 `nginx -v`，覆盖 CMD 的默认参数，输出 nginx 版本后容器退出。

#### 典型生产示例 2：自定义初始化脚本

生产环境中，通常使用 ENTRYPOINT 执行初始化脚本，完成配置渲染、环境变量加载、依赖检查等操作，然后启动业务主进程。

```dockerfile
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY app.jar /app/app.jar
COPY entrypoint.sh /app/entrypoint.sh
RUN chmod +x /app/entrypoint.sh

# 固定入口为初始化脚本
ENTRYPOINT ["/app/entrypoint.sh"]
# 默认启动参数
CMD ["java", "-jar", "app.jar"]
```

entrypoint.sh 脚本核心规范（必须以`exec "$@"`结尾，将 1 号进程替换为业务进程）：

```bash
#!/bin/bash
set -e
# 初始化操作：配置文件渲染、数据库等待、权限调整等
echo "执行容器初始化操作"
envsubst < /app/config.template.yaml > /app/config.yaml
# 执行CMD传递的命令，替换当前Shell进程为业务进程，保证业务进程是1号进程
exec "$@"
```

### 5.5 核心避坑规范

1. **严禁使用 Shell 格式的 CMD/ENTRYPOINT**，必须使用 Exec 格式，避免信号传递失效、无法优雅停止的问题；
2. **容器 1 号进程必须是业务主进程**，避免 1 号进程是 Shell、脚本等父进程，业务进程是子进程，导致生命周期绑定错误、信号传递失效；
3. **初始化脚本必须以 exec 结尾**，将 1 号进程替换为业务主进程，保证生命周期与信号传递正常；
4. **业务进程必须前台运行**：容器的 1 号进程必须在前台运行，不能以守护进程 / 后台模式启动，否则容器启动后会立即退出。
