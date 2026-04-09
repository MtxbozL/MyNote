
WebSocket 是基于 HTTP/1.1**协议升级机制**实现的全双工长连接通信协议，广泛应用于 IM、实时通知、行情推送、协同编辑等实时性要求高的场景。由于 WebSocket 是**长连接 + 状态敏感**的协议，Nginx 做反向代理时，需做特殊的协议升级、连接保持配置，否则会导致连接中断、通信异常。

### 4.5.1 WebSocket 代理的核心难点

1. **协议升级**：WebSocket 需通过 HTTP/1.1 的`Upgrade`与`Connection`请求头完成从 HTTP 到 WebSocket 的协议升级，而这两个头为**逐跳头**，默认不会被 Nginx 透传至后端；
2. **长连接保持**：WebSocket 连接需长期保持，而 Nginx 默认的`proxy_read_timeout`为 60s，若 60s 内无数据传输，Nginx 会主动断开连接；
3. **实时性要求**：WebSocket 用于实时通信，需禁用 Nginx 的代理缓冲，避免数据传输延迟。

### 4.5.2 基础 WebSocket 代理配置（核心）

Nginx 从 1.3.13 版本开始原生支持 WebSocket 代理，核心是通过`proxy_set_header`透传协议升级相关的请求头，并配置 HTTP/1.1 协议与长连接超时，以下为基础的 ws（非加密）代理配置。

```nginx
location /ws/ {
    # 转发至后端WebSocket服务
    proxy_pass http://websocket_backend/;
    # 核心：使用HTTP/1.1协议，支持协议升级
    proxy_http_version 1.1;
    # 透传Upgrade头，指定升级为WebSocket协议
    proxy_set_header Upgrade $http_upgrade;
    # 透传Connection头，指定连接升级
    proxy_set_header Connection "upgrade";
    # 透传客户端真实IP与Host头
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $host;
    # 核心：延长长连接超时时间，避免Nginx主动断连
    proxy_read_timeout 3600s;
    # 禁用缓冲，保证实时性
    proxy_buffering off;
}
# 定义后端WebSocket服务池
upstream websocket_backend {
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    # 长连接优化：设置每个Worker进程与后端的长连接数
    keepalive 10;
}
```

### 4.5.3 生产级优化配置（推荐）

基础配置中，将`Connection`头固定为`upgrade`会导致**非 WebSocket 请求被错误升级**，生产环境中需通过`map`指令实现`Connection`头的**动态赋值**：WebSocket 请求赋值为`upgrade`，普通 HTTP 请求赋值为`close`，同时支持 wss（加密 WebSocket）代理。

#### 完整生产级配置（支持 ws/wss）

```nginx
http {
    # 核心：动态配置Connection头，避免非WebSocket请求错误升级
    map $http_upgrade $connection_upgrade {
        default upgrade; # WebSocket请求：Connection=upgrade
        '' close;        # 普通HTTP请求：Connection=close
    }

    # 后端WebSocket服务池，开启IP哈希保证会话粘性
    upstream websocket_cluster {
        ip_hash; # 解决WebSocket会话共享问题
        server 192.168.1.101:8080 weight=3;
        server 192.168.1.102:8080 weight=2;
        server 192.168.1.103:8080 backup;
        keepalive 20; # 长连接数优化
    }

    server {
        # ws协议：监听80端口
        listen 80;
        server_name ws.example.com;
        location /ws/ {
            proxy_pass http://websocket_cluster/;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            # 引用动态的Connection头
            proxy_set_header Connection $connection_upgrade;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $host;
            proxy_read_timeout 86400s; # 超时设置为1天
            proxy_send_timeout 86400s;
            proxy_buffering off;
        }
    }

    server {
        # wss协议：监听443端口，开启SSL加密
        listen 443 ssl;
        server_name ws.example.com;
        # SSL证书配置
        ssl_certificate /etc/nginx/ssl/ws.example.com.crt;
        ssl_certificate_key /etc/nginx/ssl/ws.example.com.key;
        ssl_protocols TLSv1.2 TLSv1.3; # 禁用不安全的TLS版本

        location /ws/ {
            proxy_pass http://websocket_cluster/;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-Proto $scheme; # 透传加密协议
            proxy_read_timeout 86400s;
            proxy_buffering off;
        }
    }
}
```

### 4.5.4 WebSocket 代理关键优化点

1. **会话粘性**：WebSocket 是长连接，需保证同一个客户端的请求始终转发至**同一台后端服务节点**，否则会导致会话中断，需在`upstream`中配置`ip_hash`或`hash $remote_addr`实现会话粘性；
2. **超时时间**：将`proxy_read_timeout`设置为足够大的值（如 1 天），或让后端服务定期发送**WebSocket Ping 帧**重置超时，避免 Nginx 主动断开空闲连接；
3. **禁用缓冲**：必须配置`proxy_buffering off`，保证数据的实时传输，避免因缓冲导致的消息延迟；
4. **SSL 加密**：生产环境中推荐使用 wss 协议（WebSocket+SSL），防止数据在传输过程中被窃取、篡改；
5. **连接数限制**：通过`limit_conn`指令限制单个 IP 的 WebSocket 连接数，防止恶意客户端建立大量长连接占用资源。

### 4.5.5 常见问题排查

1. **连接建立失败**：检查是否配置`proxy_http_version 1.1`与`Upgrade/Connection`头透传，是否开启后端服务的 WebSocket 监听；
2. **连接频繁断开**：检查`proxy_read_timeout`是否过短，后端服务是否发送 Ping 帧，网络是否存在防火墙断开长连接；
3. **数据传输延迟**：检查是否禁用`proxy_buffering`，是否开启了 Nginx 的压缩功能（WebSocket 数据无需压缩）；
4. **会话丢失**：检查`upstream`是否配置了会话粘性（如`ip_hash`），避免请求被转发至不同的后端节点。
