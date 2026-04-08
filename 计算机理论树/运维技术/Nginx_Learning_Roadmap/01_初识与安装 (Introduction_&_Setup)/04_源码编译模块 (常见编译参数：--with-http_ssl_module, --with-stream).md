
Nginx 采用**模块化架构**，核心功能与扩展能力均由模块实现，分为核心模块、标准模块、第三方模块三类。模块仅在编译时被静态 / 动态集成至 Nginx 二进制文件中，未编译的模块不会被加载，可实现功能的最小化裁剪与性能最优适配。

### 1.4.1 基础路径类编译参数

该类参数用于定义 Nginx 的安装目录结构，所有相对路径配置均基于基础路径生效，核心参数如下：

- `--prefix=PATH`：指定 Nginx 的安装根目录，默认值为`/usr/local/nginx`，是所有相对路径的基准目录；
- `--sbin-path=PATH`：指定 Nginx 二进制可执行文件的存储路径；
- `--conf-path=PATH`：指定 nginx.conf 主配置文件的默认路径；
- `--error-log-path=PATH`：指定全局错误日志的默认存储路径；
- `--pid-path=PATH`：指定 pid 文件存储路径，用于记录 Master 主进程的 PID；
- `--lock-path=PATH`：指定 lock 文件存储路径，用于实现资源互斥锁。

### 1.4.2 核心标准模块编译参数

标准模块为 Nginx 官方提供的功能模块，部分核心模块默认不启用，需通过编译参数手动指定，生产环境高频使用的核心参数如下：

- `--with-http_ssl_module`：启用 HTTP SSL 模块，支持 HTTPS 协议与 TLS 加密传输，依赖 OpenSSL 库，是生产环境 HTTPS 部署的必选参数；
- `--with-stream`：启用四层 TCP/UDP 代理模块，实现 TCP/UDP 协议的反向代理、负载均衡与流量转发，可用于 MySQL、Redis 等中间件的代理部署，是企业级四层负载的必选参数；
- `--with-http_stub_status_module`：启用状态监控模块，提供 Nginx 运行状态、并发连接数、请求处理数等核心指标，是监控场景的必选参数；
- `--with-http_realip_module`：启用真实 IP 透传模块，用于反向代理场景下获取客户端真实 IP 地址，反向代理部署必选；
- `--with-http_gzip_module`：启用 HTTP Gzip 压缩模块，默认启用，实现静态资源的压缩传输，降低带宽占用；
- `--with-http_rewrite_module`：启用 URL 重写模块，依赖 PCRE 库，默认启用，实现 Rewrite 路由重写、URL 跳转等能力。

### 1.4.3 模块裁剪与第三方模块参数

1. **模块裁剪参数**：通过`--without-http_*_module`参数禁用指定的 HTTP 标准模块，用于裁剪不必要的功能，减小二进制文件体积，提升运行安全性，适配嵌入式、等保合规等场景。
2. **第三方模块集成参数**：
    
    - `--add-module=PATH`：静态编译第三方模块，将第三方模块源码编译进 Nginx 二进制文件，模块随 Nginx 启动自动加载；
    - `--add-dynamic-module=PATH`：动态编译第三方模块，生成独立的.so 动态库文件，可在配置文件中通过`load_module`指令动态加载 / 卸载，无需重新编译整个 Nginx。
