
跨域是指**浏览器因同源策略限制**，拒绝执行来自不同域名、端口、协议的请求，是前后端分离架构中最常见的问题。Nginx 作为网关，可通过配置**CORS（跨域资源共享）** 相关的响应头，实现跨域请求的放行，无需修改前端与后端代码，是解决跨域问题的**最简单、最高效**的方式。

### 4.6.1 CORS 核心原理

CORS 通过**浏览器与服务器的协商机制**实现跨域访问，核心是服务器在响应头中添加`Access-Control-*`系列头，告诉浏览器该资源允许跨域访问的域名、方法、头等信息。浏览器会根据请求类型分为**简单请求**与**复杂请求**，处理逻辑不同：

1. **简单请求**：满足请求方法为 GET/POST/HEAD，且自定义请求头仅为简单头（如 Content-Type 为 application/x-www-form-urlencoded），浏览器直接发送请求，服务器返回带 CORS 头的响应；
2. **复杂请求**：非简单请求（如 PUT/DELETE 方法、带自定义头、发送 JSON 数据），浏览器会先发送**OPTIONS 预检请求**，验证服务器是否允许该跨域请求，预检通过后才会发送真实请求。

### 4.6.2 核心 CORS 响应头说明

CORS 的核心是配置`Access-Control-*`系列响应头，Nginx 通过`add_header`指令实现，各头的核心作用与配置规则如下：

|响应头|核心作用|配置示例|
|---|---|---|
|Access-Control-Allow-Origin|允许跨域访问的域名，* 表示允许所有域名，也可指定具体域名|* 或 [https://front.example.com](https://front.example.com)|
|Access-Control-Allow-Methods|允许跨域的 HTTP 请求方法|GET, POST, PUT, DELETE, OPTIONS|
|Access-Control-Allow-Headers|允许跨域的自定义请求头|Content-Type, Authorization, Token|
|Access-Control-Allow-Credentials|是否允许跨域请求携带 Cookie / 认证信息，值为 true/false|true|
|Access-Control-Expose-Headers|允许前端通过 JS 读取的响应头|Content-Length, X-Token|
|Access-Control-Max-Age|预检请求的缓存时间，单位为秒，缓存期间无需重复发送预检请求|1728000（20 天）|

### 4.6.3 基础跨域配置（允许所有域名）

适合开发环境或公开的 API 服务，允许所有域名的跨域请求，配置简洁，需同时处理**真实请求**与**OPTIONS 预检请求**。

```nginx
location /api/ {
    proxy_pass http://backend_service/;
    # 核心CORS头配置，always表示对所有状态码生效
    add_header Access-Control-Allow-Origin * always;
    add_header Access-Control-Allow-Methods * always;
    add_header Access-Control-Allow-Headers * always;
    add_header Access-Control-Expose-Headers "Content-Length, X-Token" always;

    # 处理OPTIONS预检请求，直接返回204无内容
    if ($request_method = 'OPTIONS') {
        add_header Access-Control-Allow-Origin *;
        add_header Access-Control-Allow-Methods *;
        add_header Access-Control-Allow-Headers *;
        add_header Access-Control-Max-Age 1728000;
        add_header Content-Length 0;
        add_header Content-Type text/plain;
        return 204;
    }
}
```

**关键说明**：`add_header`指令默认仅对 200、201、301 等成功状态码生效，添加`always`参数后，对**所有状态码**（如 404、500）均生效，避免跨域请求在服务异常时被浏览器拦截。

### 4.6.4 生产级跨域配置（允许指定域名 + 携带凭证）

生产环境中，为保证安全性，**禁止使用通配符 ***，需指定具体的允许跨域的前端域名，同时若需支持跨域请求**携带 Cookie / 认证头**（如 Token），需满足两个条件：`Access-Control-Allow-Origin`为具体域名，且`Access-Control-Allow-Credentials`为`true`。

#### 完整生产级配置示例

```nginx
location /api/ {
    proxy_pass http://backend_service/;
    # 核心：允许指定前端域名跨域，支持携带凭证
    add_header Access-Control-Allow-Origin $http_origin always;
    add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
    add_header Access-Control-Allow-Headers "Content-Type, Authorization, X-Token" always;
    add_header Access-Control-Allow-Credentials true always;
    add_header Access-Control-Expose-Headers "Content-Length, X-Token, X-Request-Id" always;

    # 处理OPTIONS预检请求
    if ($request_method = 'OPTIONS') {
        add_header Access-Control-Allow-Origin $http_origin;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS";
        add_header Access-Control-Allow-Headers "Content-Type, Authorization, X-Token";
        add_header Access-Control-Allow-Credentials true;
        add_header Access-Control-Max-Age 1728000;
        add_header Content-Length 0;
        add_header Content-Type text/plain;
        return 204;
    }

    # 反向代理基础配置
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $host;
    proxy_http_version 1.1;
}
```

**核心优化**：使用`$http_origin`变量代替固定域名，自动匹配客户端请求的 Origin 头，实现**动态允许指定域名跨域**，同时保证安全性。

### 4.6.5 跨域配置注意事项

1. **通配符与凭证冲突**：若配置`Access-Control-Allow-Credentials true`，则`Access-Control-Allow-Origin`**不可使用通配符 ***，必须为具体域名，否则浏览器会拒绝该响应；
2. **预检请求处理**：必须单独处理 OPTIONS 请求，直接返回 204，避免预检请求被转发至后端服务，导致处理异常；
3. **always 参数必加**：确保 CORS 头在所有状态码下均返回，避免服务异常时跨域请求被拦截；
4. **避免多层 CORS 配置**：若后端服务已配置 CORS 头，Nginx 无需重复配置，否则会导致响应头重复，引发浏览器异常；
5. **HTTPS 跨域**：若前端为 HTTPS 协议，后端也必须为 HTTPS 协议，浏览器禁止从 HTTPS 域名向 HTTP 域名发起跨域请求。

### 4.6.6 跨域配置验证

配置完成后，可通过**curl 命令**测试 OPTIONS 预检请求与真实请求，验证 CORS 头是否正确返回：

```bash
# 测试OPTIONS预检请求
curl -X OPTIONS -H "Origin: https://front.example.com" -H "Access-Control-Request-Method: POST" -I http://api.example.com/api/
# 测试真实GET请求
curl -H "Origin: https://front.example.com" -I http://api.example.com/api/user
```

若响应头中包含配置的`Access-Control-*`头，则说明跨域配置生效。

## 本章小结

本章围绕 Nginx 反向代理与网关的核心能力，详细讲解了企业级反向代理的全量配置与优化：首先明确了正向代理与反向代理的核心区别，厘清了二者的应用场景与价值；其次深入解析了`proxy_pass`的核心拼接规则，解决了反向代理中最易踩坑的路径拼接问题；接着讲解了客户端真实 IP 的透传方案，包括单层与多层代理的配置原则，保证后端服务能获取到真实的请求来源；随后介绍了代理配置的优化策略，涵盖超时、缓冲、长连接等核心配置，提升代理服务的稳定性与性能；然后针对 WebSocket 协议，讲解了从基础到生产级的代理配置，解决了长连接、协议升级、会话粘性等核心问题；最后讲解了 CORS 跨域配置，实现了前后端分离架构中跨域问题的高效解决，无需修改业务代码。

本章内容是 Nginx**企业级应用的核心**，反向代理作为 Nginx 最常用的功能，是微服务网关、负载均衡、统一入口等架构的基础，需重点掌握`proxy_pass`的拼接规则、真实 IP 透传、WebSocket 代理的核心配置、CORS 跨域的生产级配置，同时理解代理优化的核心思路，根据业务场景做精细化配置。后续的负载均衡、缓存机制等进阶功能，均基于反向代理的核心配置展开。