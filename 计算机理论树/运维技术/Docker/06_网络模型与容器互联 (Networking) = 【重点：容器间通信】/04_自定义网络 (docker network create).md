
自定义网络是用户通过`docker network create`命令自行创建的 Docker 网络，底层默认使用 bridge 驱动，完全兼容 Bridge 模式的虚拟化架构，同时彻底解决了默认网桥的所有核心缺陷，是 Docker 官方明确推荐的生产环境唯一网络方案。

### 4.1 自定义网络的全生命周期管理

Docker 提供完整的网络管理命令，遵循`docker network <子命令>`的管理式语法规范，覆盖自定义网络的全生命周期管控。

|命令|核心作用|典型生产用法|
|---|---|---|
|`docker network create`|创建自定义网络|`docker network create --driver bridge --subnet 172.20.0.0/16 --gateway 172.20.0.1 --label env=prod prod_bridge`|
|`docker network ls`|列出宿主机所有 Docker 网络|`docker network ls --filter driver=bridge --filter label=env=prod`|
|`docker network inspect`|查看网络完整元数据，包括网段、网关、接入容器、IP 分配等|`docker network inspect prod_bridge`|
|`docker network connect`|将运行中的容器动态接入指定网络|`docker network connect prod_bridge nginx`|
|`docker network disconnect`|将容器从指定网络中移除|`docker network disconnect prod_bridge nginx`|
|`docker network rm`|删除指定自定义网络，仅可删除无容器接入的网络|`docker network rm prod_bridge`|
|`docker network prune`|批量清理无容器接入的未使用网络|`docker network prune --filter "until=24h" --filter "label=env=test"`|

#### 核心创建参数生产规范

- `--driver/-d`：指定网络驱动，单主机环境默认使用`bridge`，多主机集群场景可使用`overlay`驱动；
- `--subnet`：**生产环境必须显式指定**，自定义网络的 CIDR 网段，提前规划避免与宿主机、机房内网、云平台网段冲突；
- `--gateway`：指定网络的网关 IP，必须属于`--subnet`指定的网段；
- `--ip-range`：指定容器 IP 的分配范围，实现精细化的地址管理；
- `--label`：为网络添加业务、环境标签，便于分类管理、过滤与审计；
- `--opt`：配置网桥高级参数，如 MTU、网桥名称、iptables 规则等，适配生产环境的网络规范。

#### 容器接入自定义网络的两种方式

1. **创建容器时静态指定**：`docker run -d --network prod_bridge --name nginx nginx:1.27.0`，容器创建时直接接入自定义网络，分配对应网段的固定 IP 规则；
2. **运行中容器动态接入**：`docker network connect prod_bridge mysql`，无需重启容器，动态将运行中的容器接入目标网络，容器会获得多网卡多 IP 地址，实现跨网络通信。

### 4.2 核心优势 1：内置 DNS 名称解析（核心推荐原因）

自定义网络内置了**Docker 嵌入式 DNS 解析服务**，接入同一自定义网络的容器，可直接通过**容器名 / 主机名**互相通信，无需硬编码 IP 地址，彻底解决了默认网桥的核心缺陷。

#### 底层实现原理

1. Docker 为每个自定义网络运行一个独立的嵌入式 DNS 解析器，监听在容器内的`127.0.0.11`地址；
2. 容器创建时，Docker 自动修改容器内的`/etc/resolv.conf`文件，将 DNS 服务器设置为`127.0.0.11`，同时配置对应的搜索域；
3. 当容器通过容器名发起通信时，嵌入式 DNS 解析器会直接解析出对应容器的 IP 地址，容器间直接通过该 IP 通信；
4. 若解析的域名非容器名，DNS 解析器会自动转发到宿主机的 DNS 服务器，实现公网域名的正常解析。

#### 核心生产价值

1. **彻底解耦 IP 与通信链路**：容器重启、重建后 IP 地址可能变更，但容器名固定，通信链路完全不受影响，完美适配容器的动态生命周期特性；
2. **原生适配微服务架构**：多容器微服务部署时，服务之间可直接通过服务名（容器名）通信，无需额外部署服务发现组件，大幅简化部署架构；
3. **配置可移植性极强**：容器间的通信配置完全不依赖宿主机环境，可跨主机、跨环境无缝复用，完全符合 Docker「一次构建，到处运行」的设计理念。

### 4.3 核心优势 2：精细化网络隔离与管控

1. **业务级二层隔离**：不同自定义网络之间默认完全隔离，属于不同网络的容器无法直接通信，即使网段重叠也互不影响，实现不同业务、不同环境的网络隔离，大幅缩小攻击面，符合最小权限原则；
2. **独立配置管理**：每个自定义网络可独立配置网段、网关、MTU、iptables 规则，配置变更仅影响接入该网络的容器，无需重启 Docker Daemon，不影响其他业务；
3. **精细化流量管控**：可针对单个自定义网络配置带宽限制、流量监控、防火墙规则，实现业务级别的网络管控，满足生产环境的合规审计要求。

### 4.4 跨网络通信标准化实现

不同自定义网络之间默认完全隔离，若需实现跨网络的容器通信，Docker 提供两种标准化实现方案，**严禁通过关闭防火墙、手动修改 iptables 规则等方式强行打通**，避免破坏 Docker 的网络隔离体系。

#### 方案 1：容器多网络接入（官方唯一推荐）

通过`docker network connect`命令，将需要跨网络通信的容器，同时接入多个需要互通的自定义网络，容器会获得每个网络的 IP 地址，可直接与对应网络内的容器通信。

- 典型场景：Nginx 反向代理容器需要同时接入前端业务网络与后端数据库网络，实现前端请求转发，同时保证前端与后端网络完全隔离；
- 操作示例：

    ```bash
    # 创建两个隔离的业务网络
    docker network create frontend_net
    docker network create backend_net
    # 启动Nginx容器，接入前端网络
    docker run -d --name nginx --network frontend_net nginx:1.27.0
    # 启动MySQL容器，接入后端网络
    docker run -d --name mysql --network backend_net -e MYSQL_ROOT_PASSWORD=123456 mysql:8.0
    # 将Nginx容器动态接入后端网络，实现跨网络通信
    docker network connect backend_net nginx
    ```
    
- 核心优势：无需修改网络全局配置，仅为需要通信的容器开通跨网络访问权限，粒度精细，符合最小权限原则，安全性最高，是官方推荐的跨网络通信方案。

#### 方案 2：自定义 iptables 转发规则

适用于需要整个网络段互通的场景，通过在宿主机 iptables 的 FORWARD 链添加精细化的转发规则，实现两个自定义网络网段之间的互通，生产环境需严格限制源 / 目标网段，避免过度开放权限。

### 4.5 生产环境最佳实践

1. **强制使用自定义网桥**：生产环境所有业务容器必须接入用户自定义的网桥网络，严禁使用默认 docker0 网桥，符合 Docker 官方安全与工程化规范；
2. **按业务维度划分网络**：不同业务、不同环境（生产 / 测试 / 开发）、不同安全等级的服务，创建独立的自定义网络，实现网络隔离，缩小攻击面；
3. **显式规划网段**：创建自定义网络时，必须显式指定`--subnet`与`--gateway`，提前完成全环境 IP 地址规划，避免网段冲突；
4. **容器名通信规范**：同一网络内的容器通信，必须使用容器名 / 服务名，严禁硬编码容器 IP 地址，适配容器的动态生命周期；
5. **最小权限跨网通信**：跨网络通信仅为必要的容器开通多网络接入权限，严禁将整个网络段打通，避免过度开放；
6. **定期资源清理**：定期清理未使用的自定义网络，避免无效的网桥、iptables 规则占用宿主机资源，保障网络性能。
