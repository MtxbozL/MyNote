
## 5.4 健康检查机制

健康检查是负载均衡高可用的核心保障，用于实时检测后端节点的服务可用性，自动隔离故障节点，恢复后自动重新加入调度，避免将请求分发至故障节点导致用户请求失败。Nginx 提供两种健康检查模式：**内置被动健康检查（开源版默认支持）** 与**主动健康检查（第三方模块 / 商业版支持）**。

### 5.4.1 内置被动健康检查（开源版默认）

- **核心原理**：基于真实用户请求的事后检测机制，Nginx 仅在**请求转发过程中**检测节点可用性，若请求节点失败，根据`max_fails`与`fail_timeout`规则标记节点为不可用，实现故障隔离。无独立的探测请求，完全基于真实用户流量检测。
- **核心依赖**：`max_fails`、`fail_timeout`、`proxy_next_upstream`三个指令配合实现，三者缺一不可。
- **完整生产级配置示例**：
    
    nginx
    
    ```
    http {
        upstream backend_app {
            server 192.168.1.101:8080 max_fails=3 fail_timeout=10s;
            server 192.168.1.102:8080 max_fails=3 fail_timeout=10s;
        }
    
        server {
            listen 80;
            server_name api.example.com;
            location /api/ {
                proxy_pass http://backend_app/;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                # 核心：定义请求失败的判定条件
                proxy_next_upstream error timeout invalid_header http_502 http_503 http_504;
                # 最多重试2个节点，避免重试风暴导致集群雪崩
                proxy_next_upstream_tries 2;
                # 10秒内仅允许1次重试
                proxy_next_upstream_timeout 10s;
            }
        }
    }
    ```
    
- **proxy_next_upstream 核心参数说明**：
    
    - `error`：与后端节点建立连接、发送请求、读取响应时发生网络错误；
    - `timeout`：连接超时、发送超时、读取超时；
    - `http_502/503/504`：后端返回对应的 HTTP 状态码；
    - `invalid_header`：后端返回无效的响应头。
    
- **优缺点分析**：
    
    - 优点：开源版原生支持，无需额外编译模块，配置简单，无额外的探测请求开销；
    - 缺点：基于真实用户请求检测，故障节点会导致部分用户请求失败；无法提前发现故障节点，只能在请求失败后隔离；无法检测节点的业务可用性（如接口返回 200 但业务逻辑异常）。
    

### 5.4.2 主动健康检查（第三方模块 / 商业版）

- **核心原理**：事前检测机制，Nginx 独立启动**定时探测任务**，按照指定的时间间隔，向后端节点发送预设的探测请求（如`GET /health`），根据探测响应判断节点可用性，提前隔离故障节点，无需等待真实用户请求失败。
- **实现方式**：
    
    1. 商业版 Nginx Plus 原生支持`ngx_http_upstream_hc_module`模块；
    2. 开源版 Nginx 需编译淘宝开源的第三方模块`nginx_upstream_check_module`（业界主流方案）。
    
- **核心优势**：提前发现故障节点，无用户请求失败；可自定义探测接口、请求方式、判定规则，支持业务可用性检测；故障隔离与恢复完全自动化，不影响用户体验。
- **开源版编译与完整配置示例**：
    
    1. 编译要求：源码编译 Nginx 时，通过`--add-module=./nginx_upstream_check_module`添加第三方模块；
    2. 生产级配置：
    
    nginx
    
    ```
    http {
        upstream backend_app {
            server 192.168.1.101:8080;
            server 192.168.1.102:8080;
            # 核心：主动健康检查配置
            check interval=3000 rise=2 fall=3 timeout=2000 type=http;
            check_http_send "GET /health HTTP/1.1\r\nHost: api.example.com\r\nConnection: close\r\n\r\n";
            check_http_expect_alive http_2xx;
        }
    
        # 健康检查状态页面，用于监控与告警
        server {
            listen 8080;
            location /status {
                check_status;
                allow 192.168.1.0/24; # 仅允许内网访问
                deny all;
            }
        }
    
        server {
            listen 80;
            server_name api.example.com;
            location /api/ {
                proxy_pass http://backend_app/;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
            }
        }
    }
    ```
    
- **核心参数说明**：
    
    - `interval=3000`：探测间隔，单位毫秒，每 3 秒发送一次探测请求；
    - `rise=2`：连续 2 次探测成功，标记节点为可用，重新加入调度；
    - `fall=3`：连续 3 次探测失败，标记节点为不可用，隔离节点；
    - `timeout=2000`：探测请求超时时间，单位毫秒，超时视为探测失败；
    - `type=http`：探测类型，支持 http、tcp、ssl_hello、mysql 等；
    - `check_http_send`：定义探测请求的 HTTP 报文内容；
    - `check_http_expect_alive`：定义探测成功的响应状态码，此处 2xx 视为成功。
    
- **最佳实践**：探测接口使用轻量级的健康检查接口（如`/health`），避免业务逻辑过重导致探测超时；探测间隔推荐 3~5 秒，平衡故障发现速度与节点压力；配合监控系统采集健康检查状态指标，实现故障告警。