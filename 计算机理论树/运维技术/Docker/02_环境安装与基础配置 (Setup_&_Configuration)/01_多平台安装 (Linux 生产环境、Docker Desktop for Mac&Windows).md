## 本章导读

本章是 Docker 实操体系的前置基础章节，核心目标是完成 Docker 环境的标准化部署、合规性配置与全链路可用性验证，覆盖开发测试环境与 Linux 生产环境的全场景部署规范，同时深入讲解 Docker 引擎核心配置的底层逻辑与生产级最佳实践。本章所有操作均为后续镜像、容器、编排等实操内容的前置依赖，需完整掌握配置原理与验证方法，确保环境稳定性与合规性。

## 2.1 多平台安装

Docker 的部署分为两大场景：**Linux 生产环境部署 Docker Engine**（企业生产环境唯一推荐方案）、**桌面端开发测试环境部署 Docker Desktop**（适配 Mac/Windows 系统），二者的底层架构、适用场景与安装规范存在本质差异，需严格区分。

### 2.1.1 前置系统要求

Docker 的核心运行依赖 Linux 内核的 Namespace、Cgroups 机制与 Overlay2 存储驱动，因此对运行环境有明确的底层要求：

- **Linux 系统**：内核版本≥3.10，强烈推荐≥4.15 LTS 版本，需开启 overlay2 文件系统支持；主流适配发行版为 Ubuntu 20.04+/Debian 11+/RHEL 8+/CentOS Stream 8+/Fedora 35+；
- **Windows 系统**：Windows 10 21H2+/Windows 11，需开启 WSL2（Windows Subsystem for Linux 2）后端，或启用 Hyper-V 功能；
- **Mac 系统**：macOS 12+，适配 Intel 芯片与 Apple Silicon 全系列芯片。

### 2.1.2 Linux 生产环境安装（Docker Engine）

Docker Engine 是 Docker 的核心服务端组件，无图形化界面，仅提供 CLI 与 API 接口，是企业生产环境的标准部署方案，官方推荐通过软件仓库安装，便于后续版本管理与安全更新。

#### 2.1.2.1 Debian/Ubuntu 系列发行版安装步骤

1. **安装前置依赖包**

    ```bash
    sudo apt-get update
    sudo apt-get install ca-certificates curl gnupg lsb-release apt-transport-https
    ```
    
2. **添加 Docker 官方 GPG 密钥，验证软件包完整性**

    ```bash
    sudo mkdir -m 0755 -p /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    ```
    
3. **添加 Docker 官方稳定版软件源**

    ```bash
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```
    
4. **安装 Docker Engine 核心组件**

    ```bash
    sudo apt-get update
    sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    ```
    
    组件说明：
    
    - `docker-ce`：Docker 社区版守护进程核心包
    - `docker-ce-cli`：Docker 客户端 CLI 工具包
    - `containerd.io`：符合 OCI 标准的容器运行时
    - `docker-buildx-plugin`：镜像构建扩展插件，支持多阶段构建与多架构镜像构建
    - `docker-compose-plugin`：Docker Compose 编排工具插件
    
5. **生产环境版本锁定**
    
    为避免自动更新导致的兼容性故障，生产环境需锁定已安装的 Docker 版本：

    ```bash
    sudo apt-mark hold docker-ce docker-ce-cli containerd.io
    ```
    

#### 1.2.2 RHEL/CentOS/Fedora 系列发行版安装步骤

1. **安装前置依赖包**

    ```bash
    sudo dnf -y install dnf-plugins-core
    ```
    
2. **添加 Docker 官方稳定版软件源**

    ```bash
    sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    ```
    
3. **安装 Docker Engine 核心组件**

    ```bash
    sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    ```
    
4. **生产环境版本锁定**
    
    ```bash
    sudo dnf versionlock docker-ce docker-ce-cli containerd.io
    ```
    

#### 1.2.3 通用配置与安全说明

1. **启动 Docker 服务并设置开机自启**

    ```bash
    sudo systemctl enable docker --now
    ```
    
2. **非 root 用户管理 Docker 配置**
    
    可将指定用户加入`docker`用户组，避免每次执行 docker 命令都需使用 sudo：

    ```bash
    sudo usermod -aG docker $USER
    ```
    
    **安全警示**：`docker`组用户对`/var/run/docker.sock`拥有读写权限，等价于间接获取宿主机 root 权限，生产环境需严格控制`docker`组用户范围，禁止非授权用户加入。

### 1.3 Docker Desktop for Mac/Windows 安装

Docker Desktop 是面向桌面端开发测试环境的一体化产品，内置了 Docker Engine、Docker CLI、Docker Compose、Kubernetes 单节点集群与图形化管理界面，通过轻量级 Linux 虚拟机实现 Docker 运行环境（Mac/Windows 系统无 Linux 内核，无法直接运行 Docker Daemon）。

#### 1.3.1 安装规范

1. **前置环境准备**
    
    - Windows 系统：需在「启用或关闭 Windows 功能」中开启「适用于 Linux 的 Windows 子系统」与「虚拟机平台」，完成 WSL2 内核更新后重启主机；
    - Mac 系统：无需额外前置配置，确保系统版本符合要求即可。
    
2. **安装流程**
    
    从 Docker 官方网站下载对应系统架构的 Docker Desktop 安装包，按照图形化向导完成安装，安装完成后启动应用，即可自动完成 Docker 环境的初始化，无需手动配置服务自启。

#### 1.3.2 核心差异说明

Docker Desktop 仅适用于开发测试环境，**严禁用于企业生产环境**，核心原因包括：

1. 底层依赖虚拟机，存在额外的性能损耗，无法达到 Linux 原生 Docker Engine 的性能表现；
2. 内置组件多，攻击面大，不符合生产环境最小化部署的安全规范；
3. 商业授权限制，企业规模使用需购买商业许可，而 Docker Engine 社区版完全免费开源。
