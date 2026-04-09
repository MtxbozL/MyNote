
## 03 访问控制（IP 黑白名单）

Nginx 的 IP 访问控制能力由**ngx_http_access_module**标准模块提供，无需额外配置即可启用，核心功能是基于客户端 IP 地址实现黑白名单管控，允许 / 拒绝特定 IP 或 IP 段的访问请求，是服务端第一层网络防护屏障。

### 3.1 核心指令与语法

|指令|官方完整语法|合法作用域|核心功能|补充说明|
|:--|:--|:--|:--|:--|
|`allow`|`allow address \| CIDR \| unix: \| all;`|http、server、location、limit_except|放行匹配规则的客户端访问请求|1. 支持单个 IP、CIDR 网段、Unix 套接字、全量`all`四种匹配方式<br><br>2. 规则按**从上到下顺序匹配**，命中即停止，不会继续向下匹配|
|`deny`|`deny address \| CIDR \| unix: \| all;`|http、server、location、limit_except|拒绝匹配规则的客户端访问请求，默认返回`403 Forbidden`|1. 语法、匹配规则与`allow`完全一致<br><br>2. 最常用的黑白名单写法：先`allow`指定信任网段，最后用`deny all;`拒绝其余所有访问|

#### 3.1.1 地址格式支持

1. 完整 IPv4 地址：如`192.168.1.100`
2. IPv4 CIDR 网段：如`192.168.1.0/24`、`10.0.0.0/8`
3. 完整 IPv6 地址：如`2001:0db8::1`
4. IPv6 CIDR 网段：如`2001:0db8::/32`
5. 关键字`all`：代表所有 IP 地址

### 3.2 匹配规则与执行逻辑

1. **匹配顺序**：同作用域内，`allow`与`deny`指令按配置文件的书写顺序自上而下匹配，**匹配到第一条符合条件的规则后立即终止匹配，不再执行后续规则**。
2. **作用域继承规则**：子作用域未配置任何 access 规则时，会继承父作用域的规则；子作用域配置了 access 规则时，会完全覆盖父作用域的规则，不再继承。
3. **默认行为**：无任何匹配规则时，默认允许所有 IP 访问；仅配置`allow`规则未配置兜底`deny`时，默认允许所有未匹配的 IP 访问。

### 3.3 标准配置示例

#### 3.3.1 全局黑白名单配置

```nginx
http {
    # 全局访问控制：仅允许内网网段访问，拒绝所有其他IP
    allow 10.0.0.0/8;
    allow 172.16.0.0/12;
    allow 192.168.1.0/24;
    deny all;
}
```

#### 3.3.2 后台管理系统精准访问控制

```nginx
server {
    listen 443 ssl http2;
    server_name admin.example.com;
    # 证书配置省略

    # 全局允许办公网IP，拒绝其他IP
    allow 223.xxx.xxx.xxx/32;
    allow 114.xxx.xxx.xxx/28;
    deny all;

    # 超管登录接口额外放开专线IP
    location /api/super/login {
        # 子作用域配置覆盖父级，仅允许专线IP访问
        allow 106.xxx.xxx.xxx/32;
        deny all;
        proxy_pass http://admin_backend;
    }
}
```

#### 3.3.3 特定路径禁止外部访问

```nginx
server {
    listen 443 ssl http2;
    server_name www.example.com;
    root /var/www/html;

    # 禁止外部访问.env、.git等敏感文件
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    # 仅允许内网访问phpMyAdmin
    location /phpmyadmin/ {
        allow 192.168.1.0/24;
        deny all;
    }
}
```

### 3.4 关键注意事项

1. **反向代理场景的真实 IP 获取**：当 Nginx 处于 CDN、负载均衡等反向代理节点之后时，`allow/deny`默认获取的是代理节点的 IP，而非客户端真实 IP，需通过`real_ip`模块透传真实 IP：

    ```nginx
    http {
        # 配置信任的代理节点IP段
        set_real_ip_from 10.0.0.0/8;
        # 从X-Forwarded-For头中获取客户端真实IP
        real_ip_header X-Forwarded-For;
        # 递归排除信任的代理IP，获取最原始的客户端IP
        real_ip_recursive on;
    }
    ```
    
2. **规则顺序陷阱**：必须遵循「精准规则在前，泛规则在后」的原则，禁止将`deny all;`放在规则开头，否则会导致后续所有`allow`规则完全失效。
3. **IPv6 适配**：生产环境需同时配置 IPv4 与 IPv6 的访问规则，避免 IPv6 网络环境下的访问控制失效。
4. **日志记录**：拒绝访问的请求默认会记录 error_log，可通过`access_log off;`关闭无效访问的日志，减少磁盘 IO 开销。

---
