# 第五章 负载均衡 (Load_Balancing)

负载均衡是分布式架构中解决服务单点瓶颈、提升系统并发承载能力、实现服务高可用的核心技术。Nginx 通过内置的`ngx_http_upstream_module`（七层负载均衡）与`ngx_stream_module`（四层负载均衡），实现了灵活、高性能的流量调度能力，是企业级服务集群的核心流量入口组件。本章围绕 Nginx 负载均衡的全量核心内容展开，涵盖 Upstream 模块基础、负载均衡调度算法、后端节点状态管理、健康检查机制、四层负载均衡五大核心模块，完整覆盖生产环境负载均衡的配置、优化与高可用实践。

## 5.1 Upstream 模块（定义后端服务池）

Upstream 模块是 Nginx 负载均衡的核心载体，用于定义一组后端服务节点（上游服务器）组成的服务池，Nginx 将客户端请求按照指定的调度算法分发至池内的可用节点，实现流量的负载分发与故障隔离。

### 5.1.1 语法与作用域

- 核心语法：`upstream 服务池名称 { ... }`
- 作用域：**仅能在 http 块（七层负载）或 stream 块（四层负载）中定义**，不可在 server、location 块中配置，定义完成后可在多个 location/proxy_pass 中复用。
- 核心特性：服务池名称全局唯一，支持自定义调度算法、节点状态管理、健康检查、长连接优化等全局配置。

### 5.1.2 基础配置示例

七层 HTTP 负载均衡的最小可用配置如下，完整覆盖服务池定义、反向代理转发、基础透传配置：

nginx

```
http {
    # 定义后端应用服务池，名称为backend_app
    upstream backend_app {
        # 后端节点配置格式：server 地址[:端口] [状态参数];
        server 192.168.1.101:8080;
        server 192.168.1.102:8080;
        server 192.168.1.103:8080;
    }

    server {
        listen 80;
        server_name api.example.com;
        location /api/ {
            # 反向代理至定义的服务池，实现负载均衡
            proxy_pass http://backend_app/;
            # 反向代理基础配置（复用第四章核心规则）
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_connect_timeout 30s;
            proxy_read_timeout 60s;
        }
    }
}
```

### 5.1.3 Upstream 块核心全局指令

该类指令定义在 upstream 块内、server 节点指令之外，用于配置服务池的全局调度、性能、状态共享规则：

1. **keepalive**：配置 Nginx Worker 进程与后端服务之间的长连接缓存数量，减少 TCP 连接建立 / 销毁的开销，提升高并发下的转发性能。语法：`keepalive 连接数;`，示例：`keepalive 32;`，需配合`proxy_http_version 1.1`与`proxy_set_header Connection ""`使用。
2. **zone**：定义共享内存区域，让所有 Worker 进程共享服务池的配置与运行时状态，实现健康检查、状态监控的全局一致性。语法：`zone 区域名称 内存大小;`，示例：`zone backend_zone 64k;`，生产环境推荐开启。
3. **调度算法指令**：`ip_hash`、`least_conn`、`hash`等，用于指定服务池的负载均衡调度算法，未配置时默认使用加权轮询算法。