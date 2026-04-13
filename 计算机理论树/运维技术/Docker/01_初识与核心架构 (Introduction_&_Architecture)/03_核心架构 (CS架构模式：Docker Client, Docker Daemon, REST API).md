
Docker 采用**Client-Server（C/S，客户端 - 服务器）** 架构设计，整体架构分为三大核心组件，各组件职责边界清晰，通过标准化接口实现交互，完整架构如下：

### 3.1 Docker Client（Docker 客户端）

Docker Client 是用户与 Docker 系统交互的唯一入口，核心职责是接收用户指令并转发至 Docker Daemon，同时向用户返回执行结果。

- 主流实现方式为 Docker CLI 命令行工具，用户通过`docker run`、`docker build`、`docker pull`等指令完成所有 Docker 操作；
- 同时支持 Docker SDK、REST API 直接调用等方式，适配自动化脚本、二次开发等场景；
- 客户端本身不执行任何容器管理操作，仅作为指令转发的终端，可与 Docker Daemon 运行在同一主机，也可通过网络实现远程主机的 Docker Daemon 访问。

### 3.2 Docker Daemon（Docker 守护进程）

Docker Daemon（进程名为`dockerd`）是 Docker 架构的核心控制平面与执行引擎，是运行在宿主机后台的常驻服务。

- 核心职责是持续监听并处理来自 Docker Client 的请求，完成镜像、容器、网络、数据卷等所有 Docker 资源的生命周期管理；
- 作为上层控制组件，其本身不直接执行容器的底层创建与运行操作，而是通过标准化接口调用下层的 containerd、runc 组件，完成容器的底层管控；
- 支持通过配置文件调整运行参数，实现镜像加速器、存储路径、日志驱动、私有仓库信任等核心能力的配置。

### 3.3 REST API

REST API 是 Docker Client 与 Docker Daemon 之间通信的标准化接口，基于 HTTP/HTTPS 协议实现 RESTful 风格的接口规范。

- 该 API 定义了 Docker 所有操作的标准化请求与响应格式，覆盖镜像管理、容器生命周期、网络配置、数据卷管理等全量功能；
- 是 Docker 架构解耦的核心，客户端与守护进程的所有交互均通过该 API 完成，同时支持第三方运维平台、自动化工具通过该 API 对接 Docker 系统，实现容器管理的二次开发与自动化调度。
