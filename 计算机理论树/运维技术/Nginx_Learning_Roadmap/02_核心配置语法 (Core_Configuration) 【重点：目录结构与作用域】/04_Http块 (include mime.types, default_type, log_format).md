
Http 块是`main`全局块的直接子块，是 Nginx**HTTP/HTTPS 服务的核心配置块**，所有与 HTTP 协议相关的配置均在此块中定义，可包含多个`server`块，其配置指令会被所有子`server`块继承，是实现 HTTP 协议通用配置、模块化管理的关键，修改后需通过`reload`热重载生效。

### 2.4.1 核心配置指令

1. **include mime.types;**
    
    - 作用：引入 MIME 类型映射文件，定义文件扩展名与 HTTP 响应头`Content-Type`的对应关系，决定浏览器如何解析返回的静态资源（如.html 文件解析为网页，.jpg 文件解析为图片）；
    - 配套指令：`default_type application/octet-stream;`，当请求的文件扩展名未在`mime.types`中定义时，默认返回的 MIME 类型，浏览器会将其视为二进制文件下载；
    - 注意：`mime.types`文件默认位于 Nginx 配置目录，修改后需重载配置，且不支持通配符扩展名，新增类型需明确列出。
    
2. **log_format**
    
    - 语法：`log_format 格式名 '格式化字符串';`
    - 作用：定义 HTTP 访问日志的自定义格式，支持嵌入 Nginx 内置变量，实现日志内容的个性化定制，是日志分析、监控的基础；
    - 核心要求：**必须在 Http 块顶层定义**，不可在 server 块或 location 块中定义，所有子块可通过格式名引用；
    - 示例：`log_format main '$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent"';`；
    - 配套指令：`access_log 日志路径 格式名;`，指定访问日志存储路径并引用自定义日志格式，可在 http/server/location 块中多层覆盖。
    
3. **sendfile**
    
    - 语法：`sendfile on | off;`
    - 作用：开启 Linux 零拷贝技术，跳过内核缓冲区与用户缓冲区之间的数据拷贝，直接将磁盘文件发送至网络套接字，大幅提升静态资源传输性能；
    - 示例：`sendfile on;`（生产环境推荐开启，是静态资源优化的核心指令）；
    - 配套指令：`tcp_nopush on;`，开启后在 sendfile 模式下，将数据一次性发送至客户端，减少网络包数量，提升传输效率。
    
4. **keepalive_timeout**
    
    - 语法：`keepalive_timeout 超时时间 [header超时时间];`
    - 作用：设置 HTTP 长连接的超时时间，即一个 TCP 连接在多次请求之间的空闲保持时间，减少 TCP 连接建立与关闭的开销；
    - 示例：`keepalive_timeout 65s;`（客户端与 Nginx 的长连接空闲 65 秒后关闭）；
    - 注意：超时时间不宜过长，否则会导致服务器持有大量空闲连接，占用资源；也不宜过短，否则无法发挥长连接优势。
    
5. **client_max_body_size**
    
    - 语法：`client_max_body_size 大小;`
    - 作用：限制客户端上传请求的最大体大小，防止超大文件上传导致的服务器资源耗尽；
    - 单位：支持`k`（千字节）、`M`（兆字节）、`G`（吉字节），示例：`client_max_body_size 100M;`；
    - 注意：若客户端上传文件超过该限制，Nginx 会返回 413 Request Entity Too Large 错误。
    

### 2.4.2 核心子块与模块化管理

1. **upstream 块**：在 Http 块内定义，用于配置负载均衡的后端服务池，为所有 server 块提供后端服务地址复用，是负载均衡的核心配置；
2. **include 指令**：通过`include conf.d/*.conf;`引入外部 server 块配置文件，将不同站点的配置拆分至独立文件，实现配置的模块化管理，提升可维护性；
3. **map/geo 块**：在 Http 块内定义，基于内置变量实现值映射（如 IP 地址映射、请求头映射），为个性化路由、访问控制提供支持。
