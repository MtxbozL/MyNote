#### 核心标准答案

1. **Nginx：同时支持四层代理和七层代理**
    
    - 七层代理：是 Nginx 最常用的能力，基于`ngx_http_module`模块实现，工作在 OSI 模型的应用层，可解析 HTTP/HTTPS/HTTP2 等应用层协议，实现基于域名、URL、请求头、Cookie 等内容的流量转发、负载均衡、反向代理、SSL 终结、缓存、限流等高级功能。
    - 四层代理：Nginx 通过`ngx_stream_module`模块（1.9.0 版本之后内置）实现，工作在 OSI 模型的传输层，可基于 TCP/UDP 协议进行流量转发和负载均衡，无需解析应用层协议，可代理 MySQL、Redis、SSH 等 TCP 服务，以及 DNS 等 UDP 服务。
    
2. **LVS（Linux Virtual Server，Linux 虚拟服务器）：仅支持四层代理**
    
    LVS 工作在 OSI 模型的传输层，是 Linux 内核内置的四层负载均衡技术，基于内核的 netfilter 框架实现，仅能解析 TCP/UDP 协议的四层信息（源 IP、目的 IP、源端口、目的端口），实现流量的转发和负载均衡，无法解析应用层协议，不支持七层代理能力。

补充：四层代理（传输层）核心是基于 IP + 端口进行流量转发，不感知应用层内容；七层代理（应用层）核心是解析应用层协议内容，基于内容做更精细的流量调度。

#### 专家级拓展

- 核心配置示例（面试可直接说出，体现实操经验）：
    
    1. Nginx 七层代理配置示例：
        
        nginx
        
        ```
        http {
            upstream backend_server {
                server 192.168.1.10:8080;
                server 192.168.1.11:8080;
            }
            server {
                listen 80;
                server_name example.com;
                # 基于URL的七层转发
                location /api/ {
                    proxy_pass http://backend_server;
                    proxy_set_header Host $host;
                    proxy_set_header X-Real-IP $remote_addr;
                }
            }
        }
        ```
        
    2. Nginx 四层代理配置示例：
        
        nginx
        
        ```
        stream {
            upstream mysql_backend {
                server 192.168.1.20:3306;
                server 192.168.1.21:3306;
            }
            server {
                listen 3306;
                proxy_pass mysql_backend;
                # TCP超时配置
                proxy_connect_timeout 3s;
                proxy_timeout 30s;
            }
        }
        ```
        
    
- LVS 的三种核心工作模式：
    
    1. DR 模式（直接路由）：最常用的生产模式，仅修改数据帧的 MAC 地址，请求经过 LVS，响应直接从后端服务器返回给客户端，不经过 LVS，性能极高，可支撑千万级并发，要求 LVS 和后端服务器在同一个二层网络。
    2. NAT 模式：网络地址转换模式，请求和响应都经过 LVS，LVS 做源 / 目的 IP 地址转换，可跨网段部署，但性能受限于 LVS 的带宽，适合小规模场景。
    3. TUN 模式（IP 隧道）：通过 IP 隧道封装数据包，支持跨机房、跨网段部署，响应不经过 LVS，性能接近 DR 模式，适合异地多机房集群。
    
- 生产环境标准架构：通常采用「LVS + Nginx」的两级负载均衡架构，LVS 作为第一层四层负载均衡，承接公网的海量并发流量，分发到多个 Nginx 节点；Nginx 作为第二层七层反向代理，做基于域名、URL 的精细流量转发，分发到后端的应用服务，兼顾了 LVS 的高性能和 Nginx 的灵活七层能力。

#### 面试避坑指南

- 严禁说「Nginx 只能做七层代理，不能做四层代理」，这是面试最高频的错误，很多新手不知道 Nginx 的 stream 模块，会被直接判定为对 Nginx 的理解不全面；
- 避免说「LVS 可以做七层代理」，LVS 工作在内核态的传输层，无法解析应用层协议，绝对不支持七层代理，这是原则性错误；
- 不要只回答能不能，不说对应的模块、工作层级、适用场景，面试问这个问题，核心是考察你对四层和七层代理的理解，以及实际落地能力。