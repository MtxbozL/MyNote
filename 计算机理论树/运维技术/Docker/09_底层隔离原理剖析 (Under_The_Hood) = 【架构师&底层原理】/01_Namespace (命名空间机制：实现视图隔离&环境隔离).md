## 本章导读

本章是 Docker 技术体系的底层原理核心章节，面向架构师级深度认知，**揭示 Docker 容器技术的本质：基于 Linux 内核原生机制实现的进程级隔离与资源管控**，彻底解答「Docker 容器到底是什么？」「和虚拟机的本质差异是什么？」两个核心问题。

本章内容完全承接前序所有章节的实操逻辑，所有 Docker 命令、配置、功能的底层实现，均基于本章讲解的三大内核基石：**Namespace（视图隔离）、Cgroups（资源限制）、UnionFS（文件系统封装）**。本章内容偏学术向，严格遵循 Linux 内核技术规范，无遗漏覆盖所有核心隔离机制，同时是容器相关面试的最高频深度考点。

### 前置核心定义：容器的本质

Docker 容器并非轻量级虚拟机，不存在任何硬件级虚拟化，其本质是**宿主机上一组被 Linux 内核三大机制严格管控的特殊进程集合**：

1. **Namespace**：为进程构建独立的全局系统资源视图，实现环境隔离，让进程「以为」自己运行在独立的操作系统中；
2. **Cgroups**：对进程组进行统一的资源配额管控，限制 CPU、内存、IO 等资源的最大使用量，防止进程抢占宿主机资源；
3. **UnionFS**：为进程构建独立的根文件系统视图，实现镜像分层与可写层隔离，保证运行环境的一致性。

三者协同工作，共同实现了容器的隔离性、可移植性与资源可控性，构成了 Docker 技术的完整底层基石。

## 01 Namespace (命名空间机制：实现视图隔离 / 环境隔离)

Linux Namespace 是内核提供的**全局系统资源隔离机制**，核心设计思想是：将原本 Linux 系统中全局可见的系统资源，封装到独立的 Namespace 中，每个 Namespace 拥有独立的资源实例，Namespace 内的进程仅能看到当前 Namespace 内的资源，对其他 Namespace 的资源完全不可见、不可访问，从而实现进程级的环境隔离。

Namespace 是 Docker 容器「隔离性」的核心实现，容器与宿主机、容器与容器之间的所有环境隔离，均由不同类型的 Namespace 协同完成。

### 1.1 Namespace 核心操作 API 与生命周期

Linux 内核为 Namespace 提供了 3 个核心系统调用，是所有容器运行时的底层操作接口，学术层面必须掌握其核心能力：

|系统调用|核心作用|Docker 中的应用场景|
|---|---|---|
|`clone()`|创建新进程，并为其创建新的 Namespace，将新进程加入该 Namespace|容器创建的核心入口，`docker run`时通过 clone 系统调用创建容器的 1 号进程，并初始化所有隔离 Namespace|
|`setns()`|将当前进程加入一个已存在的 Namespace|`docker exec`的底层实现，将新创建的 Shell 进程加入目标容器的所有 Namespace，实现进入容器的效果|
|`unshare()`|将当前进程移出当前 Namespace，创建并加入新的 Namespace|容器运行时的动态隔离配置，以及底层调试场景|

#### Namespace 生命周期规则

每个 Namespace 的生命周期由其内部的进程数量决定：当 Namespace 内的所有进程都终止时，该 Namespace 会被内核自动销毁。Docker 通过`/proc/[pid]/ns/`目录下的 Namespace 文件绑定挂载，实现 Namespace 的生命周期与容器绑定，即使容器内无进程运行，也可保留 Namespace 实例。

### 1.2 六大隔离维度（Docker 全量启用）

Docker 为每个容器默认创建并启用 6 种独立的 Namespace，实现全维度的环境隔离，覆盖容器运行所需的所有系统资源视图，无任何遗漏。

#### 1.2.1 PID Namespace（进程隔离）

- **内核支持版本**：Linux 2.6.24
- **核心隔离能力**：隔离进程 ID 空间，每个 PID Namespace 拥有独立的 PID 编号体系，不同 Namespace 的 PID 可以重复，实现进程树的完全隔离。
- **核心特性与 Docker 实现**：
    
    1. **层级化结构**：PID Namespace 采用父子层级架构，父 Namespace 可以看到子 Namespace 内的所有进程，并对其进行管控；子 Namespace 无法看到父 Namespace 及兄弟 Namespace 的任何进程，Docker 中宿主机为顶层父 Namespace，每个容器为独立的子 Namespace；
    2. **1 号进程特权**：每个 PID Namespace 内的第一个进程（PID 1）是该 Namespace 的 init 进程，内核会为其赋予特殊权限：负责回收 Namespace 内的僵尸进程，同时接收所有未被捕获的信号；容器的 1 号进程就是镜像 ENTRYPOINT/CMD 启动的业务进程，其生命周期完全决定了容器的生命周期，对应第四章容器生命周期的核心规则；
    3. **进程可见性限制**：容器内通过`ps`、`top`命令仅能看到当前 PID Namespace 内的进程，无法看到宿主机及其他容器的进程，对应`docker top`命令的底层实现：在宿主机的顶层 Namespace 中，读取容器 PID Namespace 内的进程信息并展示。
    
- **安全边界**：PID Namespace 彻底阻断了容器内进程通过 PID 访问、干扰宿主机及其他容器进程的路径，是容器进程隔离的核心屏障。

#### 1.2.2 NET Namespace（网络隔离）

- **内核支持版本**：Linux 2.6.29
- **核心隔离能力**：隔离网络协议栈资源，每个 NET Namespace 拥有完全独立的网络栈，包括：网卡设备、路由表、iptables/netfilter 规则、端口空间、socket 套接字、网络连接跟踪表、协议栈参数。
- **核心特性与 Docker 实现**：
    
    1. **完全独立的网络环境**：默认 Bridge 模式下，Docker 为每个容器创建独立的 NET Namespace，容器拥有自己的`eth0`虚拟网卡、独立 IP、路由表，监听的端口仅在当前 Namespace 内生效，不同容器的端口互不冲突，彻底解决端口占用问题；
    2. **网络互通实现**：通过 veth pair（虚拟网卡对）将容器的 NET Namespace 接入宿主机的 Linux 网桥（docker0 / 自定义网桥），实现容器间、容器与宿主机的网络互通，对应第六章 Bridge 网络的底层实现；
    3. **Host 模式本质**：`--network host`参数的底层实现，是容器不创建独立的 NET Namespace，直接共享宿主机的根 NET Namespace，因此可以直接使用宿主机的所有网卡、端口、路由表，无网络虚拟化开销；
    4. **None 模式本质**：容器创建独立的 NET Namespace，但不为其创建任何虚拟网卡，仅保留本地回环接口`lo`，实现完全的网络隔离。
    
- **安全边界**：NET Namespace 实现了容器网络栈的完全隔离，默认情况下，容器内无法监听宿主机端口、无法修改宿主机路由表、无法访问其他容器的网络连接，是容器网络安全的核心屏障。

#### 1.2.3 IPC Namespace（进程间通信隔离）

- **内核支持版本**：Linux 2.6.19
- **核心隔离能力**：隔离 System V IPC 对象（共享内存、信号量、消息队列）与 POSIX 消息队列，每个 IPC Namespace 拥有独立的 IPC 标识符键值空间，不同 Namespace 的 IPC 对象完全不可见，无法跨 Namespace 进行进程间通信。
- **核心特性与 Docker 实现**：
    
    1. 容器默认创建独立的 IPC Namespace，容器内创建的共享内存、消息队列仅在当前容器内可见，无法被宿主机或其他容器访问，避免不同容器的 IPC 对象冲突与数据泄露；
    2. Docker 支持通过`--ipc`参数配置 IPC 共享模式，如`--ipc=host`共享宿主机 IPC Namespace，`--ipc=container:<name>`共享其他容器的 IPC Namespace，仅用于需要跨容器高性能 IPC 通信的特殊场景，生产环境默认禁用。
    
- **安全边界**：IPC Namespace 阻断了容器通过共享内存、消息队列等 IPC 机制泄露数据、攻击宿主机的路径，是容器进程间通信安全的核心屏障。

#### 1.2.4 MNT Namespace（挂载点隔离）

- **内核支持版本**：Linux 2.4.19
- **核心隔离能力**：隔离文件系统的挂载点视图，每个 MNT Namespace 拥有独立的挂载点列表，进程仅能看到当前 Namespace 内的挂载文件系统，对其他 Namespace 的挂载点完全不可见。
- **核心特性与 Docker 实现**：
    
    1. **容器根文件系统实现**：Docker 为每个容器创建独立的 MNT Namespace，通过`pivot_root`系统调用（比传统`chroot`更安全的根目录切换机制），将容器的根文件系统切换为镜像构建的 Overlay2 联合文件系统，容器内仅能看到镜像的目录结构与挂载的数据卷，无法看到宿主机的完整文件系统；
    2. **挂载传播规则**：MNT Namespace 定义了 4 种挂载传播类型，控制挂载事件在 Namespace 之间的传播行为，Docker 默认使用`private`私有传播模式，容器内的挂载操作仅在当前 MNT Namespace 内生效，不会传播到宿主机或其他容器，对应第五章绑定挂载、数据卷的底层实现；
    3. **与 UnionFS 的协同**：Overlay2 联合文件系统的挂载操作，仅在容器的 MNT Namespace 内生效，不同容器的可写层挂载完全隔离，互不干扰。
    
- **安全边界**：MNT Namespace 是容器文件系统隔离的核心，阻断了容器内进程访问、修改宿主机文件系统的默认路径，是容器文件系统安全的核心屏障。

#### 1.2.5 UTS Namespace（主机名与域名隔离）

- **内核支持版本**：Linux 2.6.19
- **核心隔离能力**：隔离系统的主机名（hostname）与域名（domainname），每个 UTS Namespace 拥有独立的主机名与域名配置，修改当前 Namespace 的配置不会影响其他 Namespace 或宿主机。
- **核心特性与 Docker 实现**：Docker 通过`--hostname`参数为容器设置独立的主机名，底层就是为容器创建独立的 UTS Namespace 并配置主机名，容器内的`hostname`命令仅能看到当前配置的主机名，不会修改宿主机的主机名，实现主机标识的隔离。
- **安全边界**：UTS Namespace 实现了容器主机标识的隔离，避免容器修改宿主机的主机名、域名配置，同时为容器内的应用提供独立的主机名标识，适配集群服务发现场景。

#### 1.2.6 USER Namespace（用户与用户组隔离）

- **内核支持版本**：Linux 3.8
- **核心隔离能力**：隔离用户 ID（UID）与用户组 ID（GID）的映射空间，每个 USER Namespace 可以独立配置 UID/GID 映射规则，实现 Namespace 内的 UID 与宿主机 UID 的解耦。
- **核心特性与 Docker 实现**：
    
    1. **权限降权核心机制**：USER Namespace 最核心的能力是**非特权映射**：可以将容器内的 root 用户（UID 0）映射到宿主机的非特权普通用户（如 UID 10000-65535），容器内的 root 用户在当前 Namespace 内拥有完整的 root 权限，但在宿主机上仅拥有普通用户的权限，无法对宿主机进行特权操作，彻底解决容器提权的核心安全风险；
    2. **映射规则**：Docker 通过`--userns-remap`参数配置宿主机上的用户映射范围，实现全局的 USER Namespace 启用，每个容器的 UID/GID 都会被映射到指定的非特权用户范围；
    3. **权限边界**：容器内的 root 用户仅能在当前 USER Namespace 内拥有特权，对宿主机的文件、进程、设备均无特权访问权限，即使容器被突破，也无法获取宿主机的 root 权限，是容器安全加固的最核心手段。
    
- **生产规范**：生产环境必须启用 USER Namespace，配合容器内非 root 用户运行，实现双重权限降权，最大化降低容器提权风险。
