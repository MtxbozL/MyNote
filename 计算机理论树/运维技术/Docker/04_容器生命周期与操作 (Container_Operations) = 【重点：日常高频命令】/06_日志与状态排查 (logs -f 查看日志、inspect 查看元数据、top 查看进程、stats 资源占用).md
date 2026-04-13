
日志与状态排查是生产环境容器故障定位、性能分析的核心能力，本节覆盖 Docker 原生提供的全量排障工具，需掌握每个命令的适用场景、输出解读与高级用法，建立完整的容器排障体系。

### 6.1 docker logs：业务日志查看

`docker logs`用于查看容器 1 号进程的标准输出（stdout）与标准错误（stderr）日志，是业务异常排查的首要命令。

#### 6.1.1 核心语法与高频参数

```bash
docker logs [OPTIONS] CONTAINER
```

|参数|核心作用|典型用法|
|---|---|---|
|`-f/--follow`|持续跟踪日志输出，实时刷新新增日志，类似`tail -f`|`docker logs -f nginx`|
|`--tail N`|仅输出最后 N 行日志，避免全量历史日志刷屏|`docker logs --tail 100 nginx`|
|`-t/--timestamps`|显示日志的时间戳，精准定位故障发生时间|`docker logs -t nginx`|
|`--since/--until`|按时间范围过滤日志，支持绝对时间与相对时间|`docker logs --since 1h --until 30m nginx`|

#### 6.1.2 底层原理与生产规范

- 底层逻辑：Docker 通过配置的日志驱动，收集容器 1 号进程的 stdout/stderr 流，写入对应的日志存储文件，`docker logs`读取日志驱动的内容并格式化输出；
- 核心限制：`docker logs`仅能收集输出到 stdout/stderr 的日志，若业务日志直接写入容器内的文件，该命令无法查看，需进入容器或通过数据卷挂载到宿主机查看；
- 云原生规范：**生产环境业务应用必须将日志输出到 stdout/stderr**，而非容器内的文件，符合 12 因素应用规范，便于日志集中化收集、检索与监控，避免容器内日志文件无限增长导致磁盘占满。

### 6.2 docker inspect：容器全量元数据查询

`docker inspect`用于输出容器的完整元数据，JSON 格式，覆盖容器的配置、状态、网络、挂载、资源限制等所有信息，是 Docker 排障的「万能命令」，几乎所有容器异常都可通过该命令定位根因。

#### 6.2.1 核心语法与高级用法

```bash
docker inspect [OPTIONS] CONTAINER [CONTAINER...]
```

- 核心参数：`-f/--format`，基于 Go 模板格式化输出，精准提取指定字段，避免全量 JSON 输出的信息冗余，是高频高级用法；
- 典型排障用法：
    
    1. 查看容器运行状态与退出原因：`docker inspect --format '{{.State}}' nginx`
    2. 查看容器的 IP 地址与网关：`docker inspect --format '{{.NetworkSettings.IPAddress}}' nginx`
    3. 查看容器的数据卷挂载配置：`docker inspect --format '{{.Mounts}}' nginx`
    4. 查看容器的端口映射详情：`docker inspect --format '{{.NetworkSettings.Ports}}' nginx`
    5. 查看容器的重启策略与资源限制：`docker inspect --format '{{.HostConfig}}' nginx`
    

#### 6.2.2 核心输出字段解读

|顶级字段|核心内容|排障场景|
|---|---|---|
|`State`|容器运行状态、PID、启动时间、退出码、退出原因、健康检查状态|容器异常退出、启动失败、健康检查不通过|
|`Config`|镜像、启动命令、环境变量、端口声明、用户、工作目录等配置|容器启动命令错误、环境变量缺失、配置不生效|
|`HostConfig`|端口映射、重启策略、CPU / 内存资源限制、数据卷挂载、网络模式、特权配置|资源限制不生效、端口映射冲突、挂载异常|
|`NetworkSettings`|所属网络、IP 地址、网关、MAC 地址、端口映射详情、DNS 配置|容器网络不通、端口映射不生效、DNS 解析异常|
|`Mounts`|绑定挂载、数据卷的源路径、目标路径、权限、挂载模式|数据丢失、挂载权限不足、配置文件不生效|

### 6.3 docker top：容器进程查看

`docker top`用于查看容器内运行的所有进程列表，无需进入容器，即使容器内未安装`ps`命令，也可正常查看，是排查容器内进程异常、CPU 占用过高、僵尸进程的核心工具。

- 语法：`docker top CONTAINER [ps OPTIONS]`
- 底层逻辑：在宿主机上读取容器 PID Namespace 内的进程信息，映射为宿主机的进程视图，输出内容包括：宿主机 PID、容器内 PID、运行用户、CPU 占用率、内存占用、启动命令等；
- 典型用法：
    
    1. 查看容器内所有进程：`docker top nginx`
    2. 自定义输出格式，支持 Linux ps 命令的所有参数：`docker top nginx aux`
    

### 6.4 docker stats：资源占用实时监控

`docker stats`用于实时查看容器的 CPU、内存、磁盘 IO、网络 IO、进程数量等资源占用情况，基于 Cgroups 的统计数据实现，是容器性能瓶颈分析、资源超限排查、OOM 故障定位的核心工具。

#### 6.4.1 核心语法与参数

```bash
docker stats [OPTIONS] [CONTAINER...]
```

- 核心参数：
    
    - `-a/--all`：显示所有容器，包括已停止的，默认仅显示运行中的容器；
    - `--no-stream`：仅输出一次当前资源状态，不持续刷新，适用于监控脚本与数据采集；
    - `--format`：自定义输出格式，按需展示指定指标；
    
- 核心输出字段解读：

| 字段                  | 核心含义                            | 排障作用             |
| ------------------- | ------------------------------- | ---------------- |
| `CPU %`             | 容器占用的 CPU 百分比，相对于宿主机总 CPU 核心    | 定位 CPU 占用过高的容器   |
| `MEM USAGE / LIMIT` | 容器当前内存使用量 / 配置的内存上限，默认上限为宿主机总内存 | 排查内存泄漏、OOM 故障    |
| `MEM %`             | 内存占用相对于配置上限的百分比                 | 监控内存资源使用率        |
| `NET I/O`           | 容器网络的累计接收 / 发送流量                | 排查网络带宽占用异常       |
| `BLOCK I/O`         | 容器磁盘的累计读 / 写流量                  | 排查磁盘 IO 性能瓶颈     |
| `PIDs`              | 容器内当前运行的进程数量                    | 排查进程泄漏、fork 炸弹攻击 |

### 6.5 补充排障工具：docker events

`docker events`用于实时监听 Docker Daemon 的事件流，包括容器的创建、启动、停止、暂停、销毁、OOM、健康检查状态变更等所有事件，是定位容器异常退出、自动化监控的核心工具。

- 典型用法：
    
    1. 实时监听所有事件：`docker events`
    2. 按时间范围过滤历史事件：`docker events --since 24h`
    3. 过滤指定类型事件：`docker events --filter 'type=container' --filter 'event=oom'`，监听容器 OOM 事件
    
