
Nginx 的基础状态监控由**ngx_http_stub_status_module**标准模块提供，源码编译时需添加`--with-http_stub_status_module`参数启用，yum/apt 包管理器安装的 Nginx 默认已启用。该模块可暴露 Nginx 核心运行状态指标，包括连接数、请求总数、读写状态等，是服务健康检查、性能瓶颈定位、基础监控的核心手段，无需依赖任何第三方组件。

### 4.1 官方标准语法与配置

```nginx
stub_status;
```

**合法作用域**：`server`、`location`

**核心功能**：在匹配的 location 中开启状态页面，输出 Nginx 核心运行指标。

#### 4.1.1 生产环境安全配置示例

状态页面包含服务核心运行数据，禁止公网无限制访问，必须配置 IP 白名单 + HTTP 基础认证双重防护：

```nginx
server {
    listen 443 ssl http2;
    server_name status.example.com;
    ssl_certificate /etc/nginx/ssl/example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;

    # 状态监控专属location
    location /nginx_status {
        # 1. IP白名单：仅允许内网、运维办公网IP访问
        allow 127.0.0.1;
        allow 192.168.1.0/24;
        allow 223.xxx.xxx.xxx/32;
        deny all;

        # 2. HTTP基础认证，双重防护
        auth_basic "Nginx Status - Authorized Access Only";
        auth_basic_user_file /etc/nginx/auth_basic/status.htpasswd;

        # 3. 开启状态输出
        stub_status;

        # 4. 关闭日志记录，避免无效日志
        access_log off;
    }
}
```

配置完成后，重载 Nginx，通过`https://status.example.com/nginx_status`即可访问状态页面，输出格式如下：

```bash
Active connections: 291
server accepts handled requests
 214852 214852 1054723
Reading: 6 Writing: 179 Waiting: 106
```

### 4.2 核心指标深度解析

状态页面的 7 个核心指标，覆盖 Nginx 连接层、请求层的全维度运行状态，是监控与性能分析的核心，必须精准理解每个指标的含义：

|指标名称|指标含义|监控与分析价值|
|---|---|---|
|`Active connections`|当前活跃的客户端 TCP 连接总数，包括正在读写的连接与空闲等待的长连接|反映服务当前并发连接压力，数值持续超过服务器最大承载的 80% 时，需扩容或优化|
|`accepts`|Nginx 启动以来，累计接受的客户端 TCP 连接总数|累计连接量统计，用于容量规划、业务增长分析|
|`handled`|Nginx 启动以来，累计成功处理的 TCP 连接总数|正常情况下与`accepts`完全相等；若`handled`远小于`accepts`，说明服务器资源耗尽（文件描述符、内存不足），连接被丢弃，是严重故障信号|
|`requests`|Nginx 启动以来，累计处理的客户端 HTTP 请求总数|累计请求量统计，用于 QPS 计算、业务峰值分析；HTTP 长连接场景下，一个 TCP 连接可处理多个请求，因此`requests >= handled`|
|`Reading`|当前正在读取客户端请求头的 TCP 连接数|正常情况下数值很小；数值持续偏高，说明客户端上行带宽不足、请求头过大，或遭遇慢速连接攻击|
|`Writing`|当前正在向客户端发送响应的 TCP 连接数|数值持续偏高，说明响应体过大、客户端下行带宽不足，或服务器下行带宽达到瓶颈|
|`Waiting`|当前空闲的、等待客户端新请求的长连接数，也叫`keepalive`空闲连接|开启 HTTP 长连接后，该数值为正常空闲连接，数值不为 0 说明长连接复用正常；数值为 0 说明长连接未生效，每个请求都新建 TCP 连接，需优化`keepalive_timeout`参数|

### 4.3 基于指标的核心计算与分析

1. **QPS（每秒请求数）计算**：
    
    QPS 是服务核心吞吐指标，通过两次采样的`requests`差值除以采样间隔计算：

    ```bash
    QPS = (requests2 - requests1) / (time2 - time1)
    ```
    
    生产环境监控中，通常取 15 秒 / 1 分钟的采样间隔，计算实时 QPS。
    
2. **TCP 连接复用率计算**：
    
    反映 HTTP 长连接的复用效率，复用率越高，TCP 握手开销越小，服务性能越好：
    
    ```txt
    连接复用率 = requests / handled
    ```
    
    正常业务场景下，复用率应大于 5；静态资源站点应大于 10；若复用率接近 1，说明长连接未生效，需优化`keepalive`相关配置。
    
3. **请求处理成功率计算**：
    
    反映服务连接处理的稳定性：
    
    ```txt
    连接处理成功率 = handled / accepts * 100%
    ```
    
    正常情况下应等于 100%，低于 99.9% 时需排查服务器资源瓶颈、配置错误。

### 4.4 关键注意事项与最佳实践

1. **安全防护必须到位**：状态页面禁止公网无限制开放，必须配置 IP 白名单 + 基础认证，避免服务运行状态泄露，被攻击者用于定向攻击。
2. **指标生命周期**：所有累计指标（accepts、handled、requests）均为 Nginx 启动以来的累计值，重启后清零；监控系统必须计算速率指标（如 QPS、每秒新建连接数），而非直接监控累计值。
3. **健康检查适配**：该状态接口可作为 Nginx 服务的健康检查端点，负载均衡器可通过访问该接口判断 Nginx 服务是否存活，无需额外开发健康检查接口。
4. **权限约束**：状态页面的 HTTPS 证书、认证文件必须配置正确的权限，避免出现 403/401 错误，导致监控数据采集失败。
5. **采样频率规范**：基础监控采样间隔建议为 10-15 秒，避免采样过于频繁导致 Nginx 性能损耗，同时保证异常事件的及时捕获。

---
