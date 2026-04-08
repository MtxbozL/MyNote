
## 4.2 基础反向代理（proxy_pass 配置与 URL 尾部斜杠 "/" 的拼接细节）

`proxy_pass`是 Nginx 实现反向代理的**核心指令**，用于指定后端服务的转发地址，其**URL 尾部是否带斜杠 /** 直接决定请求 URI 的拼接逻辑，配置错误会导致后端服务接收到错误的请求路径，进而返回 404 资源未找到错误，是反向代理配置中最易踩坑的知识点。

### 4.2.1 proxy_pass 核心语法

- 语法：`proxy_pass http://后端服务地址[:端口][/路径];`
- 作用域：`location`块、`if`块（不推荐）、`limit_except`块
- 核心说明：后端服务地址可配置为单个服务器地址（如`192.168.1.100:8080`）或`upstream`定义的服务池名称（如`http://backend_pool`），支持 HTTP/HTTPS 协议。

### 4.2.2 核心拼接规则（带斜杠 vs 不带斜杠）

Nginx 的`proxy_pass`路径拼接逻辑遵循**两大核心规则**，核心取决于`proxy_pass`的 URL 尾部**是否包含斜杠 /**，且与`location`的匹配规则强相关，以下为最常用的**前缀匹配 location**场景的拼接规则，是生产环境的核心配置场景。

#### 规则 1：proxy_pass URL 尾部**带斜杠 /**

- 核心逻辑：Nginx 会**丢弃 location 匹配的 URI 前缀部分**，仅将**剩余的 URI**拼接至`proxy_pass`的 URL 后，作为转发至后端的请求路径。
- 核心特征：路径重写，剥离 location 匹配的前缀，适合前后端分离架构中**后端服务监听根路径 /** 的场景，是生产环境**最推荐的写法**。
- 示例解析：
    
    nginx
    
    ```
    # location匹配前缀/api/，proxy_pass尾部带/
    location /api/ {
        proxy_pass http://192.168.1.100:8080/;
    }
    ```
    
    - 客户端请求`/api/users/1` → 后端收到`/users/1`
    - 客户端请求`/api/v1/order` → 后端收到`/v1/order`
    

#### 规则 2：proxy_pass URL 尾部**不带斜杠 /**

- 核心逻辑：Nginx 会将**客户端的完整请求 URI**（包含 location 匹配的前缀）**原样拼接**至`proxy_pass`的 URL 后，作为转发至后端的请求路径。
- 核心特征：路径原样转发，不做任何修改，**仅适用于后端服务同样监听该前缀路径**的场景，若后端无对应路径，会直接返回 404，生产环境**应尽量避免**。
- 示例解析：
    
    nginx
    
    ```
    # location匹配前缀/api，proxy_pass尾部不带/
    location /api {
        proxy_pass http://192.168.1.100:8080;
    }
    ```
    
    - 客户端请求`/api/users/1` → 后端收到`/api/users/1`
    - 客户端请求`/api/v1/order` → 后端收到`/api/v1/order`
    

### 4.2.3 特殊场景：proxy_pass URL 包含子路径

当`proxy_pass`的 URL 不仅带斜杠，还包含**自定义子路径**时，拼接逻辑与规则 1 一致：丢弃 location 匹配的前缀，将剩余 URI 拼接至`proxy_pass`的完整 URL 后。

- 示例解析：
    
    nginx
    
    ```
    location /api/ {
        proxy_pass http://192.168.1.100:8080/v1/;
    }
    ```
    
    - 客户端请求`/api/users/1` → 后端收到`/v1/users/1`
    - 客户端请求`/api/order` → 后端收到`/v1/order`
    

### 4.2.4 配置最佳实践

1. **优先使用带斜杠的写法**：生产环境中，后端服务通常监听根路径，带斜杠的写法能避免路径重复拼接导致的 404 错误，是最安全、最常用的配置方式；
2. **location 与 proxy_pass 斜杠保持一致**：若`location`匹配的前缀以`/`结尾（如`/api/`），则`proxy_pass`也建议以`/`结尾，提升配置可读性；
3. **复杂路径用 rewrite 配合**：若需对请求 URI 做重排、替换等复杂处理，先通过`rewrite`指令修改 URI，再配合`proxy_pass`转发，避免直接依赖`proxy_pass`的拼接规则；
4. **正则 location 显式指定 URI**：若`location`为正则匹配模式，`proxy_pass`的 URL 尾部的斜杠将失效，需通过`rewrite`或`$1`等捕获组显式指定转发的 URI。