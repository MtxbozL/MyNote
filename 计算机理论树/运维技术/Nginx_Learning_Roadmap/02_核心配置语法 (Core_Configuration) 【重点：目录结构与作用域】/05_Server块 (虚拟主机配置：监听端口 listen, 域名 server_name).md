
Server 块是 Http 块的子块，一个 Http 块可包含**多个 Server 块**，每个 Server 块对应一个**虚拟主机（独立站点）**，通过`listen`指令绑定 IP: 端口，通过`server_name`指令匹配客户端请求的 Host 头，实现多站点在同一台服务器的隔离部署，是 Nginx 多站点配置的核心，修改后需通过`reload`热重载生效。

### 2.5.1 核心配置指令

1. **listen**
    
    - 基础语法：`listen IP:端口;`，简化语法：`listen 端口;`（绑定所有 IP 地址）；
    - 作用：指定 Server 块监听的 IP 地址与 TCP 端口，是 Nginx 接收客户端请求的入口，多个 Server 块可监听同一端口，通过`server_name`区分；
    - 核心参数：`default_server`，设置为当前端口的兜底虚拟主机，当客户端请求的 Host 头无任何 Server 块匹配时，由该 Server 块处理；
    - 示例：`listen 80 default_server;`（监听 80 端口，且为 80 端口的兜底虚拟主机）；`listen 443 ssl;`（监听 443 端口，开启 SSL 加密，用于 HTTPS 服务）；
    - 注意：一个 IP: 端口组合仅能有一个`default_server`，未指定时 Nginx 将配置中第一个匹配该 IP: 端口的 Server 块作为兜底。
    
2. **server_name**
    
    - 语法：`server_name 域名1 域名2 ...;`
    - 作用：指定虚拟主机对应的域名，Nginx 通过匹配客户端请求的 Host 头与`server_name`，将请求路由至对应的 Server 块，支持精确匹配、通配符匹配、正则匹配三种方式；
    - 匹配优先级（从高到低）：
        
        1. **精确匹配**：域名与 Host 头完全一致，如`server_name www.example.com;`匹配`Host: www.example.com`；
        2. **通配符匹配**：仅支持前导`*`或尾随`*`，且`*`为独立令牌，如`*.example.com`匹配`api.example.com`，`www.*`匹配`www.example.net`，不支持`www.*.com`；
        3. **正则匹配**：以`~`开头的 PCRE 正则表达式，按配置顺序匹配第一个成功项，如`~^www\.(.+)$`匹配所有 www 子域名；
        
    - 示例：`server_name example.com www.example.com *.example.com;`；
    - 注意：Nginx 不区分域名大小写，且匹配发生在`listen`层之后，先确定 IP: 端口，再筛选 Server 块。
    
3. **root**
    
    - 语法：`root 目录路径;`
    - 作用：指定当前虚拟主机的静态资源根目录，Nginx 会根据该路径拼接请求 URI，查找对应的静态文件；
    - 示例：`root /usr/share/nginx/html;`，当请求`http://example.com/index.html`时，Nginx 会查找`/usr/share/nginx/html/index.html`文件；
    - 注意：可在 location 块中重定义，覆盖 server 块的 root 配置，实现不同 URI 的根目录隔离。
    
4. **index**
    
    - 语法：`index 文件名1 文件名2 ...;`
    - 作用：指定 Nginx 处理目录请求时的默认首页文件，按配置顺序查找，找到第一个存在的文件即返回；
    - 示例：`index index.html index.htm index.php;`，当请求`http://example.com/`时，Nginx 会依次查找`root`目录下的`index.html`、`index.htm`、`index.php`；
    - 注意：可在 location 块中重定义，适配不同目录的默认首页需求。
    

### 2.5.2 虚拟主机配置原则

- 同一 IP: 端口下的多个 Server 块，通过`server_name`实现域名隔离，避免匹配冲突；
- 为每个 IP: 端口配置`default_server`，处理非法 Host 头请求，提升服务安全性；
- 静态资源根目录`root`建议在 server 块中统一配置，特殊 URI 在 location 块中覆盖，保持配置一致性；
- HTTPS 服务的 SSL 配置（`ssl_certificate`、`ssl_certificate_key`）必须在对应的`listen 443 ssl`的 Server 块中配置，不可在 Http 块中全局定义。