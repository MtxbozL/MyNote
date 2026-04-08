
## 06 隐藏版本号

Nginx 默认会在 HTTP 响应头`Server`、错误页面中显示完整的版本号，攻击者可通过版本号匹配对应的 Nginx 漏洞，发起定向攻击。隐藏版本号是最基础的安全防护措施，可大幅降低服务被漏洞扫描、定向攻击的风险。

### 6.1 核心指令与语法

版本号控制核心指令为`server_tokens`，隶属于**ngx_http_core_module**标准模块，无需额外配置即可启用。

**官方语法**：`server_tokens on | off | build | string;`

**合法作用域**：`http`、`server`、`location`

**默认值**：`on`

#### 6.1.1 参数详解

表格

|参数值|核心效果|
|---|---|
|`on`|开启版本号显示，HTTP 响应头`Server`与错误页面会显示完整的 Nginx 版本号，如`Server: nginx/1.24.0`|
|`off`|关闭版本号显示，HTTP 响应头`Server`仅显示`nginx`，不显示版本号；错误页面也会隐藏版本号信息|
|`build`|显示版本号与构建信息，仅用于调试场景，不推荐生产环境使用|
|`string`|自定义`Server`头内容，仅 Nginx Plus 商业版支持，开源版不支持该参数|

### 6.2 标准配置示例

nginx

```
http {
    # 全局关闭版本号显示，所有server块继承该配置
    server_tokens off;

    server {
        listen 443 ssl http2;
        server_name www.example.com;
        # 子作用域可覆盖全局配置，单独开启/关闭
        # server_tokens on;
    }
}
```

### 6.3 进阶：完全自定义 / 隐藏 Server 响应头

开源版 Nginx 无法通过`server_tokens`指令完全删除`Server`响应头，也无法自定义内容，如需实现完全隐藏或自定义，可通过以下两种方案实现：

#### 6.3.1 方案 1：源码编译修改版本号

编译安装 Nginx 前，修改源码中的版本号定义文件，实现完全自定义 Server 头内容：

bash

运行

```
# 1. 解压Nginx源码包
tar -zxvf nginx-1.24.0.tar.gz
cd nginx-1.24.0
# 2. 修改核心版本号定义文件
vi src/core/nginx.h
# 修改以下宏定义，自定义内容
#define NGINX_VERSION      "1.0.0"
#define NGINX_VER          "WebServer/" NGINX_VERSION
# 3. 重新编译安装Nginx
./configure --with-http_ssl_module --with-http_v2_module
make && make install
```

#### 6.3.2 方案 2：通过第三方模块修改 / 删除 Server 头

使用`ngx_headers_more`第三方模块，可直接修改、删除`Server`响应头，无需修改源码：

nginx

```
# 编译安装时添加模块
./configure --add-module=/path/to/ngx_headers_more
# 配置完全删除Server响应头
http {
    more_clear_headers Server;
}
# 配置自定义Server响应头
http {
    more_set_headers 'Server: Custom-WebServer';
}
```

### 6.4 关键注意事项

1. **配置生效范围**：`server_tokens off`仅能隐藏版本号，无法删除`Server`响应头，攻击者仍可识别服务为 Nginx，如需完全隐藏，需使用进阶方案。
2. **错误页面适配**：自定义错误页面可彻底避免 Nginx 默认错误页面泄露版本号，即使开启`server_tokens on`，自定义错误页面也不会显示版本信息，推荐生产环境配置自定义错误页面：
    
    nginx
    
    ```
    server {
        # 自定义错误页面，完全隐藏版本信息
        error_page 403 404 500 502 503 504 /error.html;
        location = /error.html {
            root /var/www/error;
            internal;
        }
    }
    ```
    
3. **反向代理场景适配**：当 Nginx 作为反向代理时，需同时关闭后端服务的版本号显示，避免后端服务通过响应头泄露版本信息。
4. **合规性要求**：等保 2.0、PCI DSS 等合规标准明确要求禁止服务端泄露版本号信息，`server_tokens off`是合规测评的必查项。

---
