### 1. Flag 核心定位与分类

`rewrite` 指令的 flag 参数，是**ngx_http_rewrite_module 模块中控制重写后请求生命周期、处理流程、跳转边界**的核心配置，直接决定了重写后的 URI 是进入服务内部重定向、终止重写流程，还是向客户端返回标准化 HTTP 重定向响应。

根据执行逻辑与作用范围，4 个合法 flag 可严格划分为两大类，两类 flag 的行为边界完全隔离，不可混用：

|分类|包含 flag|核心行为特征|客户端感知|
|---|---|---|---|
|内部重写流程控制类|`last`、`break`|仅在 Nginx 服务内部完成 URI 重写，不向客户端发送重定向报文|无感知，地址栏 URL 无变化|
|客户端重定向类|`redirect`、`permanent`|强制向客户端返回 HTTP 重定向响应，触发客户端发起新请求|强感知，地址栏 URL 发生变更|

### 2. 内部重写流程控制类 Flag 详解

`last`与`break`是 Nginx 内部路由调度的核心标志位，二者的共性是仅完成服务端内部 URI 改写，不改变客户端请求链路；核心差异在于**是否终止当前作用域指令集、是否发起新一轮 location 匹配**，也是生产环境中最易出现配置错误的场景。

#### 2.1 `last` 标志位

##### 2.1.1 官方定义与标准化执行流程

`last` 全称 `last instruction`，官方定义为：终止当前作用域内所有 ngx_http_rewrite_module 模块指令的执行，以重写后的新 URI 为目标，发起一轮完整的 location 匹配流程。

匹配成功后，Nginx 将严格按照以下原子步骤执行，无任何额外分支：

1. 完成当前 rewrite 指令的 URI 正则匹配与字符串替换，生成重写后的新 URI；
2. 立即终止**当前所在作用域**（server/if/location）内所有后续 rewrite 模块指令（含 rewrite、set、if、return、break 等）；
3. 以重写后的新 URI 为目标，**重新遍历并执行完整的 server 块与 location 块匹配逻辑**；
4. 新匹配到的 location 块内，按配置顺序执行指令，不再回溯之前的 rewrite 规则。

##### 2.1.2 核心特性

- **作用域边界**：仅终止当前作用域的 rewrite 指令集，不限制全局 location 匹配流程，可跨 location 块完成请求调度；
- **循环约束**：触发的新一轮 location 匹配，受 Nginx 内核默认的 10 次循环上限约束，超过上限后直接向客户端返回`500 Internal Server Error`，避免死循环；
- **执行优先级**：在 server 块中使用时，优先级高于所有 location 块的原生匹配；在 location 块中使用时，会强制跳出当前 location，重新执行全局匹配。

##### 2.1.3 适用场景

- 伪静态配置：将动态 URI 重写为语义化静态 URI，需重写后匹配到后端服务处理的 location 块；
- 全局 URL 规则归一化：在 server 块中配置的全局重写规则，需重写后进入对应 location 完成后续处理；
- 多阶段路由调度：需将 URI 重写后，交由其他 location 块完成代理、缓存、权限校验等分层逻辑。

##### 2.1.4 标准配置示例

```nginx
# 示例1：server块全局伪静态配置
server {
    listen 80;
    server_name example.com;
    root /var/www/html;

    # 全局重写规则，last触发新location匹配
    rewrite ^/article/(\d+)\.html$ /index.php?module=article&id=$1 last;
    rewrite ^/user/(\d+)$ /index.php?module=user&uid=$1 last;

    # 重写后的URI匹配至此location，执行PHP解析
    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

```nginx
# 示例2：location块内跨块路由调度
server {
    listen 80;
    server_name example.com;

    location /api/v1/ {
        # 重写旧版API路径，last触发新location匹配
        rewrite ^/api/v1/(.*)$ /api/v2/$1 last;
        # 后续rewrite指令被last终止，不会执行
        rewrite ^/api/v1/test$ /test.html break;
    }

    # 重写后的URI匹配至此，执行反向代理
    location /api/v2/ {
        proxy_pass http://backend_server;
        proxy_set_header Host $host;
    }
}
```

#### 2.2 `break` 标志位

##### 2.2.1 官方定义与标准化执行流程

`break` 全称 `break instruction`，官方定义为：终止当前作用域内所有 ngx_http_rewrite_module 模块指令的执行，锁定请求处理作用域，不发起任何新的 location 匹配。

匹配成功后，Nginx 将严格按照以下原子步骤执行，无任何额外分支：

1. 完成当前 rewrite 指令的 URI 正则匹配与字符串替换，生成重写后的新 URI；
2. 立即终止**当前所在作用域**内所有后续 rewrite 模块指令，完成 URI 重写；
3. **绝对不发起新一轮 location 匹配**，即使重写后的 URI 完全匹配其他 location 规则，也不会发生跳转；
4. 继续在**当前作用域内**，执行重写后的 URI 对应的非 rewrite 模块指令（root、alias、proxy_pass、expires 等）。

##### 2.2.2 核心特性

- **作用域锁定**：完全锁定请求的处理范围，不会跳出当前 location/server 块，无跨块调度能力；
- **无循环风险**：不触发新的 location 匹配，完全规避 rewrite 循环死循环问题，配置稳定性更高；
- **指令边界清晰**：仅终止 rewrite 模块指令，不影响当前作用域内其他模块的指令执行顺序。

##### 2.2.3 适用场景

- 静态资源内部重定向：重写后的 URI 在当前 location 的 root/alias 目录下可直接处理，无需二次匹配；
- 固定反向代理场景：重写 URI 后直接在当前 location 内执行 proxy_pass，无需跳转其他 location；
- 安全隔离场景：锁定请求处理范围，避免重写后的 URI 匹配到权限宽松的 location，引发越权访问风险。

##### 2.2.4 标准配置示例

```nginx
# 示例1：静态资源路径重写
server {
    listen 80;
    server_name example.com;
    root /var/www/static;

    location /images/ {
        # 重写URI，break锁定当前location处理
        rewrite ^/images/(.*)\.jpg$ /pic/$1.jpeg break;
        # 后续rewrite指令被终止，不会执行
        rewrite ^/images/(.*)$ /img/$1 last;
        # 继续执行当前location内的root指令，查找/var/www/static/pic/目录
        root /var/www/static;
        expires 7d;
    }
}
```

```nginx
# 示例2：反向代理URI重写
server {
    listen 80;
    server_name example.com;

    location /order/ {
        # 重写路径，break锁定当前location执行proxy_pass
        rewrite ^/order/(.*)$ /api/v1/order/$1 break;
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

#### 2.3 `last` 与 `break` 核心差异对比

|对比维度|`last` 标志位|`break` 标志位|
|---|---|---|
|核心执行行为|终止当前 rewrite 指令集，发起新一轮完整 location 匹配|终止当前 rewrite 指令集，不发起任何新的 location 匹配|
|作用域范围|可跨 location 块完成请求调度|锁定当前作用域，不支持跨块调度|
|循环风险|存在循环匹配风险，受 10 次循环上限约束|无循环风险，配置稳定性更高|
|server 块内行为|终止后续 rewrite 指令，进入正常 location 匹配流程|与 last 行为基本一致，无本质差异|
|location 块内行为|强制跳出当前 location，重新执行全局 location 匹配|始终锁定在当前 location 内完成所有指令执行|
|核心适用场景|伪静态、全局路由归一化、跨 location 调度|静态资源重写、固定反向代理、安全隔离|

### 3. 客户端重定向类 Flag 详解

`redirect`与`permanent`的核心作用是强制触发客户端 HTTP 重定向，无论`replacement`字段是否以 http/https 协议开头，都会向客户端返回标准化重定向响应，客户端地址栏 URL 会发生变更，并向重定向地址发起全新的 HTTP 请求。二者的核心差异在于**HTTP 状态码、缓存行为、SEO 权重影响**。

#### 3.1 `redirect` 标志位（302 临时重定向）

##### 3.1.1 标准化定义与执行流程

`redirect` 标志位强制返回**HTTP 302 Found**状态码（HTTP/1.0 规范中为`302 Moved Temporarily`），属于临时重定向。HTTP 规范明确：该状态码标识请求的资源临时移动至新 URI，客户端**不得缓存该重定向规则**，每次对原始 URI 的请求都需先访问源服务器，再执行跳转。

匹配成功后，标准化执行流程如下：

1. 完成正则匹配与 URI 替换，自动补全`replacement`的协议、域名、端口，生成完整的重定向 URL；
2. 立即终止当前所有 rewrite 模块指令的执行，不再处理后续任何配置；
3. 向客户端返回 HTTP 302 状态码，`Location`响应头为完整的重定向 URL；
4. 客户端收到响应后，向`Location`指定的地址发起全新的 HTTP 请求，新请求重新执行 Nginx 完整配置匹配流程。

##### 3.1.2 核心特性

- **无缓存特性**：浏览器 / 客户端默认不会缓存 302 重定向规则，服务端规则修改后，客户端下一次请求即可生效，回滚成本极低；
- **SEO 无影响**：搜索引擎不会将原始 URI 的权重、排名转移到重定向后的目标 URI，仅抓取目标 URI 内容；
- **灵活性强**：支持临时业务跳转，规则可随时调整、下线，无长期客户端影响。

##### 3.1.3 适用场景

- 临时业务跳转：网站活动页、临时维护页、节假日公告页的临时跳转；
- 登录态拦截：未登录用户临时跳转到登录页，登录后可返回原始请求地址；
- 多端适配跳转：移动端用户临时跳转到移动端站点，规则可根据业务随时调整；
- 非 SEO 相关的临时 URL 调度，无需权重传递的场景。

##### 3.1.4 标准配置示例

```nginx
# 示例1：临时活动页跳转
server {
    listen 80;
    server_name example.com;

    # 首页临时跳转到活动页，302临时重定向
    rewrite ^/$ /activity/2024_promotion.html redirect;

    # 未登录用户临时跳转到登录页
    if ($cookie_user_token = "") {
        rewrite ^/user/center/(.*)$ /login.html?redirect=$request_uri redirect;
    }
}
```

```nginx
# 示例2：移动端适配临时跳转
server {
    listen 80;
    server_name example.com;

    if ($http_user_agent ~* "mobile|android|iphone") {
        rewrite ^/(.*)$ https://m.example.com/$1 redirect;
    }
}
```

#### 3.2 `permanent` 标志位（301 永久重定向）

##### 3.2.1 标准化定义与执行流程

`permanent` 标志位强制返回**HTTP 301 Moved Permanently**状态码，属于永久重定向。HTTP 规范明确：该状态码标识请求的资源已永久移动至新 URI，客户端**应当永久缓存该重定向规则**，后续对原始 URI 的请求会直接在客户端本地完成跳转，无需再访问源服务器。

匹配成功后，执行流程与`redirect`基本一致，核心差异在于状态码与客户端缓存行为，此处仅补充差异化执行逻辑：

1. 客户端收到 301 响应后，会根据浏览器缓存策略永久存储该重定向规则，缓存周期不受服务端`Cache-Control`短周期配置限制；
2. 后续客户端对原始 URI 的请求，会直接在本地完成跳转，不会向源服务器发送请求，仅当用户清除浏览器缓存或强制刷新时，才会重新请求源服务器。

##### 3.2.2 核心特性

- **强缓存特性**：浏览器会永久缓存 301 重定向规则，即使服务端删除该规则，已访问过的客户端仍会执行跳转，回滚成本极高；
- **SEO 权重传递**：搜索引擎会将原始 URI 的域名权重、页面排名、收录量完全转移到重定向后的目标 URI，是域名迁移、URL 结构重构的行业标准方案；
- **不可逆性**：规则一旦上线，对已访问客户端具有长期影响，必须经过严格测试后方可生产环境部署。

##### 3.2.3 适用场景

- 域名永久迁移：旧域名废弃，全量流量永久切换至新域名；
- URL 结构永久重构：网站 URL 体系升级，旧 URL 永久重定向至新规范 URI；
- 全站 HTTPS 化：永久将 HTTP 请求跳转至 HTTPS 协议；
- 域名归一化：根域名与 www 域名的永久统一，解决 SEO 权重分散问题。

##### 3.2.4 标准配置示例

```nginx
# 示例1：旧域名永久迁移至新域名
server {
    listen 80;
    server_name old-domain.com www.old-domain.com;
    # 全量URI捕获，301永久重定向至新域名
    rewrite ^/(.*)$ https://www.new-domain.com/$1 permanent;
}
```

```nginx
# 示例2：HTTP永久跳转HTTPS + 根域名归一化
server {
    listen 80;
    server_name example.com www.example.com;
    # 先跳转HTTPS，再处理域名归一化
    rewrite ^/(.*)$ https://www.example.com/$1 permanent;
}

server {
    listen 443 ssl;
    server_name example.com;
    ssl_certificate /etc/nginx/ssl/example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;
    # 根域名永久跳转至www域名，归一化SEO权重
    rewrite ^/(.*)$ https://www.example.com/$1 permanent;
}
```

#### 3.3 `redirect` 与 `permanent` 核心差异对比

|对比维度|`redirect`（302 临时重定向）|`permanent`（301 永久重定向）|
|---|---|---|
|HTTP 标准状态码|302 Found|301 Moved Permanently|
|客户端缓存行为|默认不缓存，每次请求均访问源服务器|永久缓存，后续请求直接在客户端跳转|
|SEO 权重影响|不传递原始 URL 的权重与排名|完全转移原始 URL 的权重、排名与收录量|
|规则回滚成本|极低，修改后客户端立即生效|极高，已缓存的客户端需清除缓存才能生效|
|业务生命周期|临时、短期、可频繁调整|永久、长期、稳定不变|
|风险等级|低风险，配置错误影响范围可控|高风险，配置错误可能导致长期线上故障|

### 4. Flag 配置禁忌与常见陷阱

1. **单条 rewrite 指令禁止多 flag 配置**：同一条 rewrite 指令中仅可配置一个 flag，若同时配置多个 flag，Nginx 仅识别最后一个 flag，前面的 flag 会被完全忽略，导致执行逻辑完全偏离预期。
2. **location 块内 last 标志位循环陷阱**：在`location /`通用匹配块内使用 last 标志位，若重写后的 URI 无更精准的 location 匹配，会再次命中`location /`，触发循环匹配，最终超过 10 次循环上限返回 500 错误。
3. **301 永久重定向缓存陷阱**：生产环境部署 301 重定向规则前，必须在测试环境完成全量验证；禁止使用 301 规则实现临时跳转，避免错误规则被客户端永久缓存，导致无法快速回滚。
4. **break 与 proxy_pass 路径拼接误区**：在 location 内使用 break 重写 URI 后，proxy_pass 的 URL 尾部斜杠拼接规则会发生变化，需严格匹配重写后的 URI 路径，避免出现路径拼接错误。
5. **重定向 URI 捕获完整性陷阱**：使用 redirect/permanent 标志位时，必须通过正则捕获组完整保留原始 URI 的路径与参数，避免跳转后丢失请求信息；若需保留原始查询字符串，禁止在 replacement 末尾添加`?`。
6. **server 块与 location 块内 flag 行为差异**：server 块内的 last 与 break 行为基本一致，均会终止后续 rewrite 指令并进入正常 location 匹配；二者的核心差异仅体现在 location 块内，禁止混淆不同作用域的执行逻辑。

### 5. Flag 选型标准化决策流程

1. 若业务需要客户端地址栏 URL 发生变更，用户可感知跳转 → 选择客户端重定向类 flag：
    
    - 临时业务跳转、无需 SEO 权重传递、规则可能频繁调整 → 选用`redirect`
    - 永久业务变更、需要 SEO 权重传递、规则长期稳定不变 → 选用`permanent`
    
2. 若仅需服务端内部 URI 重写，客户端无感知 → 选择内部重写流程控制类 flag：
    
    - 重写后的 URI 需要交由其他 location 块处理（动态请求、跨块调度） → 选用`last`
    - 重写后的 URI 在当前 location 块内即可完成处理（静态资源、固定代理） → 选用`break`
    

### 6. 配置合规性校验规范

1. 所有 rewrite flag 配置修改后，必须执行`nginx -t`命令完成语法预校验，确认无语法错误后方可重载配置；
2. 重定向类规则上线前，必须通过`curl -I 原始URL`命令验证响应状态码、Location 头是否符合预期；
3. 301 永久重定向规则上线前，必须先使用 302 规则完成业务验证，确认逻辑无误后再切换为 301 规则；
4. 生产环境禁止配置无 flag 的 rewrite 指令，避免出现未定义的循环执行行为。