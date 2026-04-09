
HTTPS 协议的加密解密过程会带来额外的 CPU 开销与握手延迟，SSL 优化的核心目标是：**降低 TLS 握手耗时、减少服务端 CPU 开销、提升 HTTPS 访问速度，同时保障加密安全性**。本节严格覆盖目录核心优化项，并补充官方推荐的配套优化方案。

### 2.1 TLS 会话复用优化：ssl_session_cache

TLS 握手是 HTTPS 延迟的核心来源，完整 TLS 1.2 握手需要 2 次 RTT（往返时延），TLS 1.3 需要 1 次 RTT。通过会话复用机制，客户端与服务端可复用之前握手生成的会话密钥，跳过完整握手流程，将握手耗时降至 0 次 RTT。

#### 2.1.1 核心指令与语法

会话复用核心指令来自`ngx_http_ssl_module`模块：

|指令|官方语法|合法作用域|核心功能|
|---|---|---|---|
|`ssl_session_cache`|`ssl_session_cache type [size];`|`http`、`server`|配置 TLS 会话缓存的存储类型与大小|
|`ssl_session_timeout`|`ssl_session_timeout time;`|`http`、`server`|配置会话缓存的有效期，超时后缓存失效|

#### 2.1.2 缓存类型详解

|缓存类型|核心特性|适用场景|
|---|---|---|
|`off`|完全禁用会话缓存，强制每次请求执行完整握手|高安全等级场景，禁止会话复用|
|`none`|会话缓存功能关闭，但允许客户端发送会话 ID，服务端不存储，仅兼容客户端逻辑|无实际优化效果，不推荐使用|
|`builtin`|基于 OpenSSL 内置的单进程缓存，每个 Worker 进程独立存储，无法跨进程共享|单机单 Worker 进程场景，不推荐多进程使用|
|`shared`|基于共享内存的多进程共享缓存，所有 Worker 进程均可读写，是官方推荐的生产环境方案|企业级多 Worker 进程部署，性能最优|

#### 2.1.3 标准优化配置

```nginx
http {
    # 核心会话缓存配置：共享内存，16MB缓存空间，可存储约8万个会话
    ssl_session_cache shared:TLS_SESSION_CACHE:16m;
    # 会话缓存有效期：1天，超过时间强制重新握手
    ssl_session_timeout 1d;
    # 启用会话票证，兼容无状态会话复用，配合共享缓存实现跨节点复用
    ssl_session_tickets on;
    ssl_session_ticket_key /etc/nginx/ssl/ticket.key;
}
```

> 配置说明：1MB 共享内存可存储约 4000-5000 个 TLS 会话，16MB 可满足日均千万级请求的会话复用需求。

### 2.2 传输性能优化：开启 HTTP/2

HTTP/2 是 HTTP 协议的第二代标准，基于二进制帧传输，支持多路复用、头部压缩、服务端推送等核心特性，可大幅降低 HTTPS 请求的延迟，提升并发传输性能，是 HTTPS 场景的必配优化项。

#### 2.2.1 核心配置语法

HTTP/2 能力由`ngx_http_v2_module`模块提供，包管理器安装的 Nginx 默认已启用，源码编译需添加`--with-http_v2_module`参数。

**启用语法**：在`listen`指令中添加`http2`参数，仅可在启用`ssl`的监听端口配置。

```nginx
server {
    # 443端口同时启用ssl与http2，是标准配置方式
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com;
    # 证书配置省略
}
```

#### 2.2.2 配套优化配置

```nginx
http {
    # 启用HTTP/2头部压缩，减少请求头传输体积
    http2_hpack_compression on;
    # 配置HTTP/2最大并发流数量，默认128，高并发场景可调整至256
    http2_max_concurrent_streams 256;
    # 配置HTTP/2最大帧大小，平衡传输效率与延迟
    http2_max_frame_size 16384;
}
```

### 2.3 安全合规优化：禁用不安全的 TLS 版本与算法

#### 2.3.1 禁用不安全的 TLS 协议版本

TLS 1.0 与 TLS 1.1 协议存在严重的安全漏洞（如 BEAST、CRIME、POODLE 等），已被 PCI DSS 等行业合规标准禁用，主流浏览器已于 2020 年停止支持。

**标准禁用配置**：

```nginx
http {
    # 仅启用TLS 1.2与TLS 1.3，禁用所有低版本协议
    ssl_protocols TLSv1.2 TLSv1.3;
}
```

#### 2.3.2 禁用弱加密算法与套件

禁用不安全的对称加密算法（如 3DES、DES、RC4）、非对称加密算法（如 RSA 1024 位以下）、哈希算法（如 MD5、SHA1），仅启用 NIST 推荐的安全加密套件，配置见本章 01 节标准加密套件列表。

### 2.4 进阶优化方案

#### 2.4.1 OCSP Stapling 证书状态 stapling

OCSP（在线证书状态协议）用于客户端查询证书是否被吊销，默认由客户端直接向 CA 服务器查询，存在延迟与隐私泄露风险。OCSP Stapling 允许服务端提前向 CA 服务器查询证书状态，并随 TLS 握手发送给客户端，无需客户端单独查询，降低握手延迟。

**标准配置**：

```nginx
http {
    # 启用OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    # 指定CA根证书与中间证书，用于验证OCSP响应
    ssl_trusted_certificate /etc/nginx/ssl/example.com_chain.pem;
    # 配置DNS服务器，用于解析CA的OCSP服务器域名
    resolver 223.5.5.5 114.114.114.114 valid=300s;
    resolver_timeout 5s;
}
```

#### 2.4.2 HSTS 强制 HTTPS 访问

HSTS（HTTP 严格传输安全）通过响应头告知浏览器，强制使用 HTTPS 协议访问站点，即使输入 HTTP 地址，浏览器也会在本地直接跳转 HTTPS，无需经过服务端 301 跳转，防止降级劫持攻击，同时提升访问速度。

**标准配置**：

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;
    # HSTS响应头：max-age=1年，包含子域名，支持预加载
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
}
```

> 关键注意事项：`always`参数必须添加，确保非 200 状态码的响应也会携带 HSTS 头；配置前需确保全站 HTTPS 化已稳定运行，否则会导致无法访问 HTTP 站点。

---
