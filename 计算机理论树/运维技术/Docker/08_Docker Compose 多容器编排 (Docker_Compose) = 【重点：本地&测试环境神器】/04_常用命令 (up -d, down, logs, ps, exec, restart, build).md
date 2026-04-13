
Docker Compose 的所有命令均以`docker compose <子命令>`格式执行，必须在`docker-compose.yml`文件所在的目录执行，或通过`-f`参数指定配置文件路径，通过`-p`参数指定项目名。

本节按生命周期分类，覆盖所有高频核心命令，明确核心作用、常用参数与生产场景用法。

### 4.1 项目生命周期核心命令

#### 4.1.1 docker compose up

**核心作用**：创建并启动项目内的所有服务、网络、数据卷，是 Compose 最核心的命令。

- 基础语法：`docker compose up [OPTIONS] [SERVICE...]`
- 高频核心参数：

| 参数                    | 核心作用                       | 生产场景用法                                   |
| --------------------- | -------------------------- | ---------------------------------------- |
| `-d/--detach`         | 后台启动所有服务，释放当前终端，生产环境必须使用   | `docker compose up -d`                   |
| `--build`             | 启动前强制重新构建所有配置了 build 字段的服务 | 代码更新后，启动时同步构建新镜像                         |
| `--force-recreate`    | 强制重新创建所有容器，即使配置与镜像未变更      | 配置变更后强制生效，解决缓存问题                         |
| `--no-deps`           | 仅启动指定的服务，不启动其依赖的服务         | 单独重启某个服务，不影响依赖服务                         |
| `--scale SERVICE=NUM` | 扩展指定服务的副本数量，实现水平扩容         | `docker compose up -d --scale backend=3` |
| `--no-start`          | 仅创建服务容器，不启动                | 初始化环境，后续手动启动                             |

#### 4.1.2 docker compose down

**核心作用**：停止并销毁项目内的所有容器、网络，是项目销毁的核心命令，**默认不会删除命名卷与镜像**。

- 基础语法：`docker compose down [OPTIONS]`
- 高频核心参数：

| 参数                 | 核心作用                                        | 风险警示                         |
| ------------------ | ------------------------------------------- | ---------------------------- |
| `-v/--volumes`     | 同时删除项目内定义的所有命名卷，以及匿名卷                       | 生产环境严禁使用，会永久删除业务数据，仅开发测试环境可用 |
| `--rmi <type>`     | 同时删除镜像，type 可选`all`（所有镜像）、`local`（仅本地构建的镜像） | 生产环境慎用，避免镜像被误删               |
| `--remove-orphans` | 删除配置文件中未定义的孤儿容器                             | 清理项目中废弃的服务容器                 |

### 4.2 服务运维管理命令

|命令|核心作用|高频用法示例|
|---|---|---|
|`docker compose ps`|列出项目内所有服务的容器状态、端口映射、健康检查状态|`docker compose ps` 查看所有服务状态<br><br>`docker compose ps backend` 查看指定服务状态|
|`docker compose logs`|查看服务的容器日志，完全对应`docker logs`命令|`docker compose logs -f --tail 100 backend` 实时查看后端服务的最后 100 行日志<br><br>`docker compose logs -t nginx mysql` 查看多个服务的日志，带时间戳|
|`docker compose exec`|在运行中的服务容器内执行命令，完全对应`docker exec`命令，默认进入第一个容器副本|`docker compose exec -it backend /bin/bash` 进入后端服务容器的交互式终端<br><br>`docker compose exec mysql mysql -u root -p` 进入 MySQL 命令行|
|`docker compose restart`|重启项目内的指定服务或所有服务|`docker compose restart backend` 重启后端服务<br><br>`docker compose restart -t 30 nginx` 重启前等待 30 秒优雅停止|
|`docker compose start/stop`|启动 / 停止已创建的服务容器，不会重建容器|`docker compose stop backend` 停止后端服务<br><br>`docker compose start mysql` 启动已停止的 MySQL 服务|
|`docker compose kill`|强制终止服务容器，发送 SIGKILL 信号，仅紧急场景使用|`docker compose kill backend` 强制终止后端服务|
|`docker compose rm`|删除已停止的服务容器|`docker compose rm -f backend` 强制删除已停止的后端服务容器|

### 4.3 构建与镜像管理命令

|命令|核心作用|高频用法示例|
|---|---|---|
|`docker compose build`|构建配置了 build 字段的所有服务镜像，对应`docker build`命令|`docker compose build --no-cache backend` 无缓存重新构建后端服务镜像<br><br>`docker compose build --pull` 构建时拉取最新的基础镜像|
|`docker compose pull`|拉取配置文件中定义的所有服务镜像|`docker compose pull mysql redis` 仅拉取指定服务的镜像<br><br>`docker compose pull --ignore-pull-failures` 忽略拉取失败的镜像|
|`docker compose push`|推送构建好的服务镜像到配置的镜像仓库|`docker compose push backend` 推送后端服务镜像到私有仓库|

### 4.4 配置校验与排障命令

|命令|核心作用|高频用法示例|
|---|---|---|
|`docker compose config`|校验并渲染完整的配置文件，输出合并后的最终配置，是排查配置语法错误、变量替换问题的核心工具|`docker compose config` 校验配置文件语法，输出完整配置<br><br>`docker compose config --quiet` 仅校验语法，不输出内容，脚本中用于配置合法性校验|
|`docker compose top`|查看服务容器内的运行进程，对应`docker top`命令|`docker compose top mysql` 查看 MySQL 容器内的进程|
|`docker compose stats`|实时查看服务容器的资源占用情况，对应`docker stats`命令|`docker compose stats` 查看所有服务的 CPU、内存、IO 资源占用|

### 生产环境命令使用规范

1. 所有生产环境操作必须先通过`docker compose config`校验配置文件语法，确认无误后再执行变更；
2. 生产环境启动服务必须使用`-d`后台模式，严禁前台启动；
3. 执行`docker compose down`时，严禁添加`-v`参数，避免业务数据永久丢失；
4. 服务变更优先使用`docker compose up -d --build`，自动完成镜像构建、容器重建，无需手动停止服务；
5. 批量操作前，先通过指定服务名的方式测试，确认无误后再执行全量操作，避免影响整个业务。
