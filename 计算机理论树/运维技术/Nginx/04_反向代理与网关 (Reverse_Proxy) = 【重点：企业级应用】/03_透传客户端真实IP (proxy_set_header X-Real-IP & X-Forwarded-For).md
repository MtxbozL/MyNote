
Nginx 作为反向代理时，后端服务接收到的请求源 IP 为**Nginx 的服务器 IP**，而非客户端的真实 IP，导致后端服务无法记录真实的请求来源，也无法基于客户端 IP 做访问控制、日志分析、风控等操作。因此需通过`proxy_set_header`指令配置**X-Real-IP**与**X-Forwarded-For**请求头，将客户端真实 IP 透传至后端服务，是反向代理的必配项。

### 4.3.1 核心 HTTP 头说明

用于透传客户端真实 IP 的两个核心 HTTP 头为行业通用标准，后端服务可通过读取该头值获取真实 IP，二者的适用场景与取值逻辑不同：

1. **X-Real-IP**
    
    - 取值：仅存储**客户端的真实 IP 地址**，无额外信息，格式为单一 IP（如`192.168.1.10`）。
    - 适用场景：**单层反向代理**场景，架构简单，后端服务读取便捷。
    
2. **X-Forwarded-For（XFF）**
    
    - 取值：存储**请求经过的所有代理节点 IP 与客户端真实 IP**，格式为**客户端 IP, 代理 1IP, 代理 2IP,...**，第一个 IP 即为客户端真实 IP。
    - 适用场景：**多层反向代理**场景（如 CDN → Nginx → 后端服务），可完整记录请求的转发链路，是企业级架构的推荐配置。
    

### 4.3.2 单层代理场景配置（核心）

单层代理指客户端请求仅经过**一台 Nginx**转发至后端服务，是最基础的架构场景，配置简洁，可同时配置 X-Real-IP 与 X-Forwarded-For，兼顾兼容性。

#### 核心配置示例

```nginx
location /api/ {
    proxy_pass http://192.168.1.100:8080/;
    # 透传客户端真实IP至X-Real-IP
    proxy_set_header X-Real-IP $remote_addr;
    # 透传客户端真实IP与代理IP至X-Forwarded-For
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    # 透传请求协议（http/https）
    proxy_set_header X-Forwarded-Proto $scheme;
    # 透传原始请求的Host头
    proxy_set_header Host $host;
}
```

#### 核心变量说明

- `$remote_addr`：Nginx 直接接收到的 TCP 连接的源 IP，单层代理场景下即为**客户端真实 IP**；
- `$proxy_add_x_forwarded_for`：Nginx 内置组合变量，取值为`$http_x_forwarded_for, $remote_addr`，若客户端请求中无 XFF 头，则直接为`$remote_addr`。

### 4.3.3 多层代理场景配置

多层代理指客户端请求经过**多台代理服务器**（如 CDN → 前端 Nginx → 后端 Nginx → 服务），此时仅在最后一台 Nginx 配置无法获取真实 IP，需**每一层代理服务器均配置 XFF 头透传**，才能保证后端服务拿到正确的客户端真实 IP。

#### 核心配置原则

- 每一层 Nginx 均需配置`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`，实现 XFF 头的**链式拼接**；
- 仅**最外层代理**能获取到客户端真实 IP，内层代理的`$remote_addr`为上一层代理的 IP；
- 后端服务从 XFF 头中**取第一个非 unknown 的 IP**，即为客户端真实 IP。

#### 多层代理链路示例

`客户端(10.0.0.1) → CDN(1.1.1.1) → 前端Nginx(192.168.1.10) → 后端Nginx(192.168.1.20) → 后端服务(192.168.1.100)`

- CDN 转发后，XFF 头为：`10.0.0.1`
- 前端 Nginx 转发后，XFF 头为：`10.0.0.1, 1.1.1.1`
- 后端 Nginx 转发后，XFF 头为：`10.0.0.1, 1.1.1.1, 192.168.1.10`
- 后端服务读取 XFF 头，取第一个 IP`10.0.0.1`即为客户端真实 IP。

### 4.3.4 后端服务获取真实 IP 的最佳实践

后端服务读取真实 IP 时，需做**容错处理**，避免因请求头缺失、值为`unknown`导致的异常，以下为通用的读取逻辑（以 Java 为例）：

```java
public String getClientRealIp(HttpServletRequest request) {
    // 优先读取XFF头
    String ip = request.getHeader("X-Forwarded-For");
    // 处理XFF头，取第一个有效IP
    if (isValidIp(ip)) {
        String[] ipArray = ip.split(",");
        for (String ipItem : ipArray) {
            String trimIp = ipItem.trim();
            if (isValidIp(trimIp) && !"unknown".equalsIgnoreCase(trimIp)) {
                ip = trimIp;
                break;
            }
        }
        return ip;
    }
    // XFF头无效时，读取X-Real-IP
    ip = request.getHeader("X-Real-IP");
    if (isValidIp(ip)) {
        return ip.trim();
    }
    // 最后读取原生的remoteAddr（兜底）
    return request.getRemoteAddr();
}
// 校验IP是否有效
private boolean isValidIp(String ip) {
    return ip != null && !ip.trim().isEmpty() && !"unknown".equalsIgnoreCase(ip.trim());
}
```

### 4.3.5 配置注意事项

1. **开启 http_realip_module 模块**：若需在 Nginx 自身中识别客户端真实 IP（如基于真实 IP 做限流、访问控制），需在源码编译时添加`--with-http_realip_module`参数，并通过`set_real_ip_from`指定信任的代理 IP 段；
2. **避免 IP 伪造**：通过`set_real_ip_from`配置信任的代理 IP 段，仅对信任的代理节点的 XFF 头做解析，防止客户端手动伪造 XFF 头传递虚假 IP；
3. **透传核心请求头**：除 IP 相关头外，需同时透传`Host`、`X-Forwarded-Proto`等头，保证后端服务能获取到原始请求的域名、协议等信息。