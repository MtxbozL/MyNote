
本节通过三个核心操作完成 Docker 环境的全链路可用性验证，不仅验证服务是否正常运行，同时验证所有配置是否生效，深入理解 Docker 架构的交互流程。

### 4.1 docker version 命令

#### 4.1.1 核心作用

该命令用于查看 Docker Client 与 Docker Daemon 的版本信息，同时验证客户端与守护进程之间的通信是否正常，是 Docker 环境排障的首要命令。

#### 4.1.2 输出解读

命令输出分为**Client**与**Server**两大模块，核心字段说明：

- **Client 模块**：展示本地 Docker CLI 客户端的版本、API 版本、Go 编译版本、操作系统与架构等信息；
- **Server 模块**：展示 Docker Daemon 的相关信息，若该模块无输出，说明 Docker Daemon 未正常启动，或客户端无法与守护进程建立通信；
- **API Version**：Docker Client 与 Server 之间的 REST API 版本，需保证二者 API 版本兼容，否则会出现操作失败。

### 4.2 docker info 命令

#### 4.2.1 核心作用

该命令用于输出 Docker 引擎的完整系统信息、配置详情与资源状态，是验证所有`daemon.json`配置是否生效、排查环境问题的核心命令，可一次性完成全量配置的合规性校验。

#### 4.2.2 核心校验字段

执行命令后，需重点校验以下字段，确认配置生效：

- `Docker Root Dir`：Docker 存储根目录，验证存储路径修改是否生效；
- `Registry Mirrors`：配置的镜像加速器地址列表，验证加速器配置是否生效；
- `Logging Driver`：全局默认日志驱动，验证日志配置是否生效；
- `Insecure Registries`：私有仓库信任列表，验证私有仓库配置是否生效；
- `Storage Driver`：存储驱动，生产环境必须为`overlay2`，若为其他驱动需排查内核与文件系统问题；
- `Cgroup Driver`：Cgroup 驱动，若需对接 Kubernetes 集群，必须配置为`systemd`；
- `Containers`/`Images`：本地容器与镜像的数量统计，验证数据目录是否正常加载。

### 4.3 hello-world 运行流程剖析

`docker run hello-world`是 Docker 环境的全链路可用性验证操作，其完整运行流程完全对应第一章讲解的 C/S 架构与三大核心概念，可验证 Docker 环境从客户端通信、镜像拉取、容器创建到运行输出的全链路是否正常。

#### 4.3.1 完整运行流程

1. **客户端指令转发**：Docker Client 接收`docker run hello-world`指令，解析参数后，通过 REST API 将容器创建请求发送至 Docker Daemon；
2. **本地镜像查找**：Docker Daemon 接收到请求后，首先在本地存储中查找是否存在`hello-world`镜像，若本地无该镜像，进入镜像拉取流程；
3. **镜像拉取**：Docker Daemon 根据配置的镜像加速器地址，向对应的镜像仓库发起拉取请求，拉取`hello-world`镜像的最新版本，拉取完成后将镜像存储至本地存储根目录；
4. **容器创建与隔离环境初始化**：Docker Daemon 调用 containerd 组件，containerd 再向下调用 runc，通过 Linux 内核的 Namespace 机制创建六大隔离环境，通过 Cgroups 机制配置资源限制，完成容器运行环境的初始化；
5. **容器启动与命令执行**：基于`hello-world`镜像启动容器，执行镜像中预设的启动命令，输出`Hello from Docker!`提示文本；
6. **结果返回**：命令执行完成后，容器自动退出，Docker Daemon 将容器的标准输出内容通过 REST API 返回至 Docker Client，最终展示给用户。

#### 4.3.2 验证标准

若命令正常输出`Hello from Docker!`提示文本，说明 Docker 环境全链路正常，所有配置生效，具备后续所有操作的基础条件。

## 本章小结

本章完整讲解了 Docker 环境的全场景部署规范、引擎核心配置的原理与生产级最佳实践、全链路环境验证方法。通过本章学习，需掌握：

1. Docker Engine 与 Docker Desktop 的本质差异、适用场景与标准化安装流程；
2. 镜像加速器的配置原理、规范与生效验证方法；
3. Docker 引擎三大核心配置（存储路径、日志驱动、私有仓库信任）的底层逻辑、配置方法与生产环境安全规范；
4. 三大验证命令的作用、输出解读，以及`hello-world`全链路运行流程与 Docker 架构的对应关系。

本章内容为 Docker 实操的基础，需确保所有配置验证无误、环境全链路可用后，再进入后续章节的学习。