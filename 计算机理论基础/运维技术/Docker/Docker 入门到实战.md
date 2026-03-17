Docker 学习可以按**从入门到生产**的逻辑，分成 **5 个核心部分**，刚好对应面试 + 实战的完整路线：

1. **基础入门**
    
    - Docker 是什么、解决什么问题
    - 安装、启动、配置镜像源
    - 最常用命令：run、ps、exec、logs、rm、rmi 等
    
2. **镜像与容器核心**
    
    - 镜像原理：分层、UnionFS
    - 手写 Dockerfile 与构建镜像
    - 容器生命周期、资源限制（CPU / 内存）
    
3. **数据与网络**
    
    - 数据持久化：数据卷、挂载
    - 容器网络模式：bridge、host、none
    - 容器间通信
    
4. **多容器编排（必学）**
    
    - Docker Compose 单机编排
    - 用 yaml 一键启动多服务（如 Nginx+MySQL+Redis）
    
5. **高级与生产实践**
    
    - 私有镜像仓库（Harbor）
    - 日志、监控、安全基础
    - CI/CD 集成、容器化项目部署


# Docker 面试核心考点清单（直接背）

## 一、基础概念（必问）

1. Docker：轻量级容器化技术，封装应用 + 依赖，保证环境一致性
2. **容器 vs 虚拟机**
    
    - 容器：共享宿主机内核，启动快、体积小、资源占用低
    - VM：有独立 Guest OS，重、启动慢
    
3. 三大核心：镜像（只读模板）、容器（运行实例）、仓库（镜像存储）
4. 架构：C/S 架构，客户端 ↔ Docker daemon

## 二、镜像 & Dockerfile（高频）

1. 镜像：UnionFS 分层存储，复用、省空间
2. 常用命令：pull、push、images、rmi、build、history
3. Dockerfile 必考指令
    
    - FROM：基础镜像
    - RUN：构建时执行
    - CMD：启动默认命令（可被覆盖）
    - ENTRYPOINT：启动入口（不易覆盖）
    - COPY/ADD：拷贝文件（ADD 支持解压）
    - WORKDIR：工作目录
    - EXPOSE：声明端口
    
4. 镜像优化：多阶段构建、清理缓存、最小化基础镜像

## 三、容器核心（必背）

1. 生命周期：创建→启动→运行→暂停→停止→删除
2. 常用命令：run、ps、exec、logs、stop、rm、stats
3. `docker run` 关键参数
    
    - `-d` 后台运行
    - `-p` 端口映射
    - `-v` 挂载
    - `--name` 命名
    - `--restart` 重启策略
    
4. 进入容器：优先 `exec`（不中断主进程）
5. 资源限制：`--cpus`、`--memory`

## 四、数据 & 网络（重点）

### 数据持久化

1. 三种挂载：
    
    - Volume（Docker 托管，推荐）
    - Bind Mount（宿主机路径直接映射）
    - tmpfs（内存，临时）
    
2. 命令：volume create/ls/inspect/prune

### 网络

1. 四种模式：
    
    - bridge（默认，独立网络）
    - host（共享宿主机网络）
    - none（无网络）
    - container（共享其他容器网络）
    
2. 同网桥容器可通过**容器名直接通信**

## 五、Docker Compose（实战必考）

1. 作用：单机一键编排多容器（Nginx+MySQL+Redis 等）
2. yaml 核心字段
    
    - services：服务列表
    - image/build：镜像 / 构建
    - ports/volumes/networks
    - depends_on：依赖顺序
    
3. 常用命令：up、down、ps、logs、exec

## 六、生产高级（进阶问）

1. 私有仓库：Harbor 部署、权限、安全
2. CI/CD：代码→构建镜像→推送→部署
3. 安全：最小权限、非 root、漏洞扫描
4. 监控：Prometheus+Grafana、日志收集