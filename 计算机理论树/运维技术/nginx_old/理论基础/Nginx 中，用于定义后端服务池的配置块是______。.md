#### 核心结论：**upstream 块是 Nginx 中唯一专门用于定义后端服务池（上游服务集群）的配置块**，是实现反向代理、负载均衡的核心语法，这道题考的是 Nginx 反向代理的核心配置，面试 100% 会覆盖的考点。

我给你讲透 upstream 块的核心作用，帮你彻底理解：

1. upstream 块的核心定位：专门用来**集中定义一组后端服务节点，形成一个统一的服务池**，给这个池子起一个自定义名称，后续在 proxy_pass 反向代理指令中，直接引用这个名称即可，不用重复写多个后端节点。
2. 标准用法示例：

    ```nginx
    # 定义后端服务池，名称为api_backend
    upstream api_backend {
        # 池子里的多个后端服务节点，可配置权重、健康检查等
        server 192.168.1.101:8080 weight=1;
        server 192.168.1.102:8080 weight=2;
        server 192.168.1.103:8080 weight=1;
    }
    
    server {
        listen 80;
        server_name api.test.com;
        location / {
            # 直接引用服务池名称，实现负载均衡转发
            proxy_pass http://api_backend;
        }
    }
    ```
    
3. 补充误区澄清：
    - 不能填 server 块：server 块是用来定义虚拟主机的，负责监听端口、绑定域名，不负责定义后端服务池；
    - 不能填 proxy_pass：proxy_pass 只是一个转发指令，用来指定请求转发的目标，不能用来定义一组服务节点池；
    - 不能填 location 块：location 块是用来匹配 URL 路径的，只负责匹配规则，不负责定义后端服务集群。