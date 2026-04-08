
## 5.5 四层负载均衡（Stream 模块）

Nginx 的四层负载均衡基于`ngx_stream_module`模块实现，工作在 OSI 七层模型的**传输层（第四层）**，基于 IP 地址 + TCP/UDP 端口实现流量转发与负载均衡，无需解析应用层协议内容，转发性能接近硬件交换机，适用于 TCP/UDP 协议的数据库、中间件、消息队列等服务的负载均衡与高可用部署。

### 5.5.1 四层与七层负载均衡核心对比

表格

|对比维度|四层负载均衡（Stream 模块）|七层负载均衡（HTTP Upstream）|
|---|---|---|
|工作层级|OSI 传输层（TCP/UDP）|OSI 应用层（HTTP/HTTPS）|
|转发依据|源 / 目的 IP 地址、端口号|HTTP 协议内容（URI、Host、请求头等）|
|协议解析|无需解析应用层协议，仅处理 TCP/UDP 报文|需完整解析 HTTP/HTTPS 协议内容|
|转发性能|极高，接近硬件交换机转发效率|较高，低于四层负载均衡|
|核心能力|端口转发、TCP/UDP 负载均衡、高可用|路由分发、内容缓存、压缩、跨域、协议转换|
|适用场景|MySQL、Redis、MQ、TCP 长连接服务|HTTP/HTTPS API 服务、Web 服务、静态资源|

### 5.5.2 模块编译与基础配置结构

- **模块编译**：`ngx_stream_module`模块**默认不编译**，源码编译 Nginx 时，需添加`--with-stream`参数启用；如需 SSL 支持，需同时添加`--with-stream_ssl_module`参数。
- **配置结构**：Stream 块与 Http 块**完全平级**，直接定义在 main 全局块中，结构与 Http 块一致，包含 upstream 块（定义后端服务池）与 server 块（定义监听端口与转发规则）。
- **基础配置示例**：
    
    nginx
    
    ```
    # 与http块平级，四层负载均衡核心块
    stream {
        # 定义TCP服务池：MySQL主从集群
        upstream mysql_cluster {
            # 四层支持权重、max_fails、backup等所有节点参数
            server 192.168.1.101:3306 weight=2 max_fails=3 fail_timeout=10s;
            server 192.168.1.102:3306 weight=1 max_fails=3 fail_timeout=10s;
            server 192.168.1.200:3306 backup;
        }
    
        # 定义TCP服务池：Redis集群
        upstream redis_cluster {
            # 四层支持轮询、加权轮询、least_conn、hash等所有调度算法
            hash $remote_addr consistent; # 客户端IP哈希，保证会话粘性
            server 192.168.1.101:6379;
            server 192.168.1.102:6379;
        }
    
        # MySQL负载均衡监听
        server {
            listen 3306; # 监听3306 TCP端口
            proxy_pass mysql_cluster;
            # 四层代理超时配置
            proxy_connect_timeout 10s;
            proxy_timeout 300s; # 连接空闲超时时间
        }
    
        # Redis负载均衡监听
        server {
            listen 6379; # 监听6379 TCP端口
            proxy_pass redis_cluster;
            proxy_connect_timeout 10s;
            proxy_timeout 300s;
        }
    }
    
    # 原有的http块，七层负载均衡配置
    http {
        # ... 七层配置内容
    }
    ```
    

### 5.5.3 核心配置指令与特性

1. **监听指令 listen**
    
    - 语法：`listen 地址:端口 [参数];`
    - 核心参数：`udp`，启用 UDP 协议代理，默认仅监听 TCP 协议；`reuseport`，开启端口复用，提升高并发下的性能。
    - UDP 代理示例：`listen 53 udp;`（DNS 服务 UDP 代理）。
    
2. **核心代理指令**
    
    - `proxy_pass`：指定后端服务池或地址，四层代理无需加`http://`前缀，直接写服务池名称或 IP: 端口；
    - `proxy_connect_timeout`：与后端服务建立 TCP 连接的超时时间；
    - `proxy_timeout`：TCP 连接的空闲超时时间，超过该时间无数据传输，Nginx 主动断开连接；
    - `proxy_buffer_size`：TCP 数据传输的缓冲区大小，默认 4k/8k，大文件传输可适当调大。
    
3. **算法与健康检查支持**
    
    - 调度算法：四层 Stream 模块支持与七层 Http 模块完全一致的负载均衡算法，配置方式无差异；
    - 健康检查：被动健康检查原生支持，配置方式与七层一致；主动健康检查需编译`nginx_upstream_check_module`模块，支持 TCP/UDP 探测。
    

### 5.5.4 典型生产场景配置

1. **MySQL 主从读写分离**
    
    nginx
    
    ```
    stream {
        # MySQL写集群：主库
        upstream mysql_write {
            server 192.168.1.100:3306;
            server 192.168.1.101:3306 backup; # 主库故障时切换至备库
        }
        # MySQL读集群：从库
        upstream mysql_read {
            least_conn;
            server 192.168.1.102:3306 weight=3;
            server 192.168.1.103:3306 weight=2;
            server 192.168.1.104:3306 weight=1;
        }
    
        # 写端口监听3306
        server {
            listen 3306;
            proxy_pass mysql_write;
            proxy_connect_timeout 10s;
            proxy_timeout 3600s;
        }
    
        # 读端口监听3307
        server {
            listen 3307;
            proxy_pass mysql_read;
            proxy_connect_timeout 10s;
            proxy_timeout 3600s;
        }
    }
    ```
    
2. **UDP 协议 DNS 负载均衡**
    
    nginx
    
    ```
    stream {
        upstream dns_server {
            server 223.5.5.5:53;
            server 114.114.114.114:53;
        }
    
        server {
            listen 53 udp; # 核心：启用UDP协议
            proxy_pass dns_server;
            proxy_responses 1; # DNS请求对应1个响应
            proxy_timeout 5s;
        }
    }
    ```
    

### 5.5.5 配置注意事项

1. 端口占用：四层监听的端口不能与系统其他服务、七层 Http 块监听的端口冲突，否则 Nginx 启动失败；
2. 权限问题：监听 1024 以下的端口，需要 Nginx Master 进程以 root 权限启动；
3. 协议适配：TCP 服务不能配置`udp`参数，UDP 服务必须添加`udp`参数，否则代理失败；
4. 长连接优化：数据库、Redis 等长连接服务，需调大`proxy_timeout`参数，避免连接被频繁断开；
5. 安全防护：四层代理无应用层防护能力，需配合`allow/deny`指令实现 IP 黑白名单访问控制，避免未授权访问。

## 本章小结

本章完整覆盖了 Nginx 负载均衡的全量核心内容，从 Upstream 服务池的基础定义，到五大核心负载均衡调度算法的原理、配置与适用场景，再到后端节点的状态管理、故障隔离规则，随后深入解析了被动与主动两种健康检查机制的实现原理与生产级配置，最后讲解了基于 Stream 模块的四层负载均衡，涵盖 TCP/UDP 协议的代理配置与典型生产场景。

负载均衡是 Nginx 实现企业级高可用架构的核心能力，是分布式服务集群的流量入口核心，需重点掌握加权轮询、IP 哈希、最少连接三大核心算法的适用场景，故障节点的自动隔离配置，主动健康检查的生产级部署，以及四层与七层负载均衡的选型与配置。本章内容是后续高可用集群架构、性能优化的核心基础，与反向代理、缓存机制等模块深度结合，共同构成 Nginx 的企业级核心能力。