### 1. 指令官方定义与核心语法规范

`if` 指令隶属于 **ngx_http_rewrite_module** 标准 HTTP 模块，是 Nginx 中唯一内置的条件分支控制指令，无需额外编译即可启用。其核心能力是基于 Nginx 内置变量、正则表达式、文件系统状态完成条件判断，在条件成立时执行块内的 rewrite 模块指令，实现动态路由、请求过滤、参数重写等分支逻辑。

#### 1.1 标准化语法

**官方语法**：`if (condition) { ... }`

**默认配置**：无

**合法作用域**：`server`、`location`

> 严格约束：`if` 指令禁止在`main`、`events`、`http`全局块中配置，仅可在 server 块与 location 块内使用；块内需为符合 Nginx 语法规范的指令集。

#### 1.2 核心执行前提

Nginx 的`if`指令**无原生布尔类型支持**，所有条件判断均基于字符串解析与正则匹配完成，条件成立的核心判定规则为：表达式解析结果为非空、非 0 开头的字符串时，视为`true`，执行块内指令；否则视为`false`，跳过块内指令。

---

### 2. condition 条件表达式完整语法体系

条件表达式分为 5 大类，覆盖所有合法判断场景，所有语法均严格遵循 Nginx 官方规范，无任何扩展语法。

#### 2.1 纯变量判断

直接使用 Nginx 内置变量或自定义变量作为判断条件，是最基础的判断形式。

- **判定规则**：当变量值为空字符串`""`，或为以`0`开头的字符串时，视为`false`；其余所有情况均视为`true`。
- **核心约束**：无任何隐式类型转换，纯字符串级别的判定，例如变量值为`"00"`、`"0abc"`均视为`false`，变量值为`"1"`、`"false"`、`"off"`均视为`true`。
- **标准示例**：

    ```nginx
    # 判断自定义变量是否非空
    if ($custom_var) { ... }
    # 判断请求是否携带Cookie
    if ($http_cookie) { ... }
    # 判断请求Referer是否为空
    if (!$http_referer) { ... }
    ```
    
    > 注：`!` 为逻辑非运算符，可对所有条件表达式的结果取反。
    

#### 2.2 字符串等值比较判断

通过`=`（等值）与`!=`（不等值）运算符完成变量与字符串的精确匹配。

- **核心规则**：
    
    1. 仅支持字符串级别的精确匹配，**无任何数值比较能力**，即使对比内容为纯数字，也仅按字符串逐位匹配；
    2. 运算符两侧必须为变量与固定字符串，禁止双变量对比；
    3. 字符串包含空格、特殊字符时，必须用单 / 双引号包裹；
    4. 匹配区分大小写，无大小写不敏感的等值运算符。
    
- **标准示例**：

    ```nginx
    # 判断请求方法是否为POST
    if ($request_method = POST) { ... }
    # 判断域名是否为根域名
    if ($host != "www.example.com") { ... }
    # 判断响应状态码是否为404
    if ($status = "404") { ... }
    ```
    
- **严格禁忌**：禁止使用`>`、`<`、`>=`、`<=`等数值比较运算符，Nginx 官方`if`指令**完全不支持此类语法**，配置后会直接触发语法校验失败。

#### 2.3 正则匹配判断

通过正则运算符完成变量与 PCRE 正则表达式的模式匹配，是`if`指令最常用的判断形式，支持捕获组与变量引用。

|运算符|语法规则|匹配特性|
|---|---|---|
|`~`|`if ($var ~ regex) { ... }`|区分大小写的正则匹配，匹配成功视为`true`|
|`~*`|`if ($var ~* regex) { ... }`|不区分大小写的正则匹配，匹配成功视为`true`|
|`!~`|`if ($var !~ regex) { ... }`|区分大小写的正则不匹配，匹配失败视为`true`|
|`!~*`|`if ($var !~* regex) { ... }`|不区分大小写的正则不匹配，匹配失败视为`true`|

- **核心规则**：
    
    1. 正则表达式遵循 PCRE 语法规范，支持`^`、`$`锚定符、捕获组、量词等完整正则能力；
    2. 正则中包含`}`、`;`、空格等 Nginx 配置特殊字符时，必须用单 / 双引号包裹，避免配置解析错误；
    3. 匹配成功后，正则中的`()`捕获组内容会自动赋值给`$1`、`$2`…`$n`变量，`$0`代表匹配到的完整字符串，可在 if 块内全局引用；
    4. 同作用域内多次正则匹配会覆盖`$1-$n`的捕获值，仅保留最后一次匹配成功的结果。
    
- **标准示例**：

    ```nginx
    # 匹配移动端UA，不区分大小写
    if ($http_user_agent ~* "mobile|android|iphone|ipad") { ... }
    # 匹配URI并捕获文章ID，区分大小写
    if ($uri ~ ^/article/(\d+)\.html$) {
        set $article_id $1;
    }
    # 禁止非GET/POST请求方法
    if ($request_method !~ ^(GET|POST)$) {
        return 405;
    }
    ```
    

#### 2.4 文件 / 目录存在性判断

通过文件系统运算符，检查 Nginx 服务器本地文件 / 目录的状态，是静态资源服务场景的核心判断形式。

| 运算符          | 语法规则                                | 判定逻辑                                                  |
| ------------ | ----------------------------------- | ----------------------------------------------------- |
| `-f` / `!-f` | `if (-f $request_filename) { ... }` | `-f`：指定路径的文件存在视为`true`；`!-f`：文件不存在视为`true`            |
| `-d` / `!-d` | `if (-d $request_filename) { ... }` | `-d`：指定路径的目录存在视为`true`；`!-d`：目录不存在视为`true`            |
| `-e` / `!-e` | `if (-e $request_filename) { ... }` | `-e`：指定路径的文件 / 目录 / 符号链接存在视为`true`；`!-e`：均不存在视为`true` |
| `-x` / `!-x` | `if (-x $request_filename) { ... }` | `-x`：指定路径的文件具备可执行权限视为`true`；`!-x`：无执行权限视为`true`       |

- **核心规则**：
    
    1. 路径解析基于当前作用域的`root`/`alias`指令，必须使用完整绝对路径变量，禁止使用相对路径；
    2. 仅检查 Nginx 运行用户对目标路径的访问权限，若权限不足，即使文件存在也会返回`false`；
    3. 符号链接会跟随指向的目标文件 / 目录完成检查，`-e`可覆盖所有存在性判断场景。
    
- **标准示例**：

    ```nginx
    # 判断请求的静态文件不存在
    if (!-f $request_filename) {
        rewrite ^/(.*)$ /index.php?$1 last;
    }
    # 判断缓存目录存在
    if (-d /var/nginx/cache/$host) {
        set $cache_dir /var/nginx/cache/$host;
    }
    ```
    

#### 2.5 逻辑非运算符

`!` 为唯一支持的逻辑运算符，可对上述所有条件表达式的结果取反，无逻辑与`&&`、逻辑或`||`运算符，禁止多条件复合判断。

> 核心约束：Nginx 原生`if`指令不支持多条件复合判断，例如`if ($a = 1 && $b = 2)`为非法语法，会直接触发配置校验失败。多条件判断需通过`set`指令拼接变量或多级`map`映射实现。

---

### 3. 指令执行逻辑与作用域规则

#### 3.1 请求处理阶段定位

`if` 指令的执行阶段固定为 **NGX_HTTP_REWRITE_PHASE**（重写阶段），该阶段早于 location 匹配、内容处理、日志记录等所有后续阶段，执行顺序严格遵循 Nginx 请求处理生命周期：

1. `server`块内的`if`指令：在 server 块全局 rewrite 指令之后、location 匹配流程之前执行，按配置书写顺序自上而下串行执行；
2. `location`块内的`if`指令：在 location 匹配完成、location 块内 rewrite 指令之后、内容处理指令（`root`/`proxy_pass`等）之前执行，按配置书写顺序串行执行。

#### 3.2 配置继承规则

1. 上层作用域（`main`→`http`→`server`→`location`）的配置会默认继承到`if`块内；
2. `if`块内的同指令配置会覆盖上层作用域的配置，且仅在`if`条件成立时生效；
3. **核心异常**：非 rewrite 模块的指令在`if`块内的继承行为存在非预期性，是 "If is Evil" 陷阱的核心根源。

#### 3.3 分支执行约束

1. 仅当条件表达式结果为`true`时，才会执行`if`块内的指令集；
2. `if`块内的 rewrite 模块指令执行完成后，会继续执行当前作用域内的后续指令，除非遇到`last`、`break`、`return`等终止类指令；
3. 禁止嵌套`if`配置，即使语法校验通过，嵌套`if`的执行逻辑也完全不可预测，官方明确禁止该用法。

---

### 4. 核心重点："If is Evil" 陷阱深度解析

"If is Evil" 是 Nginx 官方文档明确提出的警告，核心结论为：**在 location 块内的 if 指令中使用非 rewrite 模块的指令，会导致不可预测的行为，甚至引发配置失效、服务异常**。

#### 4.1 陷阱的本质根源

Nginx 的配置是**声明式的分阶段执行模型**，而非脚本式的顺序执行模型：

1. `if` 指令隶属于 rewrite 模块，仅在`NGX_HTTP_REWRITE_PHASE`阶段执行，该阶段仅负责 URI 重写与变量赋值，不处理请求内容；
2. 当`if`块内出现非 rewrite 模块的内容处理指令（如`proxy_pass`、`root`、`add_header`、`expires`等）时，Nginx 会为`if`块创建一个独立的`content`上下文，该上下文会完全覆盖上层 location 的 content 上下文；
3. 独立上下文不会继承 location 块内的非 rewrite 模块配置，导致上层配置的`proxy_set_header`、`proxy_timeout`、`gzip`等指令完全失效，引发非预期的业务故障。

#### 4.2 高频陷阱场景与错误示例

##### 陷阱 1：location 块内 if 中使用 proxy_pass，导致请求头丢失

- **错误配置**：

    ```nginx
    location /api/ {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_pass http://127.0.0.1:8080;
    
        # 条件成立时，独立content上下文会覆盖上层配置，proxy_set_header完全失效
        if ($uri ~ ^/api/admin/) {
            proxy_pass http://127.0.0.1:8081;
        }
    }
    ```
    
- **故障表现**：匹配`/api/admin/`的请求，后端服务无法获取`Host`、`X-Real-IP`请求头，引发权限校验失败、日志记录异常。

##### 陷阱 2：if 块内使用 add_header，导致上层响应头完全丢失

- **错误配置**：

    ```nginx
    location /static/ {
        add_header Cache-Control "max-age=86400";
        root /var/www/static;
    
        # 条件成立时，仅保留if块内的add_header，上层的Cache-Control头完全丢失
        if ($request_uri ~* \.(jpg|png)$) {
            add_header X-Image-Type "static";
        }
    }
    ```
    
- **故障表现**：图片请求的响应头中无`Cache-Control`字段，浏览器缓存完全失效，引发静态资源带宽浪费。

##### 陷阱 3：使用不支持的数值比较运算符，导致配置失效

- **错误配置**：

    ```nginx
    # 非法语法，Nginx完全不支持>运算符，配置校验直接失败
    if ($request_time > 1) {
        return 504;
    }
    ```
    
- **正确替代方案**：通过正则匹配实现数值范围判断

    ```nginx
    # 匹配request_time大于等于1秒的请求
    if ($request_time ~ ^[1-9]\d*(\.\d+)?$) {
        return 504;
    }
    ```
    

##### 陷阱 4：嵌套 if 配置，导致执行逻辑完全不可预测

- **错误配置**：

    ```nginx
    location /user/ {
        if ($http_cookie ~* "user_id=([^;]+)") {
            set $user_id $1;
            # 嵌套if，执行逻辑无官方规范保障，不同版本Nginx行为不一致
            if ($user_id = "") {
                return 401;
            }
        }
    }
    ```
    

##### 陷阱 5：多条件复合判断，导致语法校验失败

- **错误配置**：

    ```nginx
    # 非法语法，Nginx不支持&&逻辑与运算符
    if ($request_method = POST && $http_content_length = 0) {
        return 400;
    }
    ```
    
- **正确替代方案**：通过 set 指令拼接变量实现多条件判断

    ```nginx
    set $condition 0;
    if ($request_method = POST) {
        set $condition "${condition}1";
    }
    if ($http_content_length = 0) {
        set $condition "${condition}2";
    }
    # 两个条件同时成立时，condition值为012
    if ($condition = "012") {
        return 400;
    }
    ```
    

---

### 5. 合法合规使用场景

Nginx 官方并未完全禁止`if`指令的使用，仅限制 location 块内非 rewrite 模块指令的滥用，以下场景为官方认可的安全使用场景，无执行逻辑异常风险。

#### 5.1 server 块内的全局条件判断

server 块内的`if`指令在 location 匹配之前执行，无 content 上下文覆盖问题，行为完全可预测，是最安全的使用场景。典型场景包括：

- 全局 HTTP 强制跳转 HTTPS；
- 域名归一化跳转（根域名→www 域名）；
- 全局 UA 拦截、非法请求过滤；
- 基于请求域名的全局路由调度。

**标准安全示例**：

```nginx
server {
    listen 80;
    server_name example.com www.example.com;

    # 全局HTTP转HTTPS，安全合规
    if ($scheme = http) {
        rewrite ^/(.*)$ https://$host/$1 permanent;
    }

    # 根域名归一化，安全合规
    if ($host = "example.com") {
        rewrite ^/(.*)$ https://www.example.com/$1 permanent;
    }

    # 拦截非法UA，安全合规
    if ($http_user_agent ~* "scanner|crawler|spider") {
        return 403;
    }
}
```

#### 5.2 location 块内仅使用 rewrite 模块指令

当 location 块内的`if`块中**仅包含 ngx_http_rewrite_module 模块的原生指令**时，执行逻辑完全确定，无上下文覆盖风险，属于安全合规用法。

rewrite 模块合法指令清单：`rewrite`、`set`、`return`、`break`、`if`（不推荐嵌套）。

**标准安全示例**：

```nginx
location /article/ {
    root /var/www/html;

    # if块内仅使用rewrite模块指令，安全合规
    if ($uri ~ ^/article/(\d+)\.html$) {
        set $article_id $1;
        rewrite ^ /index.php?module=article&id=$article_id last;
    }

    # if块内仅使用return指令，安全合规
    if ($article_id = "") {
        return 404;
    }
}
```

---

### 6. If 陷阱的官方推荐替代方案

针对绝大多数分支判断场景，Nginx 提供了原生的、无副作用的替代指令，性能更高、行为更稳定，官方优先推荐使用以下方案替代`if`指令。

#### 6.1 用 location 匹配替代路径条件判断

针对基于 URI 的分支逻辑，使用 location 匹配替代`if`正则判断，性能提升 30% 以上，无任何副作用，完全规避 if 陷阱。

|错误 if 配置|正确 location 替代方案|
|---|---|
|`nginx location / { if ($uri ~ ^/admin/) { proxy_pass http://admin_backend; } }`|`nginx location ^~ /admin/ { proxy_pass http://admin_backend; proxy_set_header Host $host; }`|

#### 6.2 用 map 指令替代多条件分支判断

`map` 指令隶属于`ngx_http_map_module`模块，在 http 块内定义，基于变量映射生成新变量，无 if 的阶段执行问题，支持多条件匹配，是官方推荐的多条件判断替代方案。

**标准替代示例**：

```nginx
# http块内定义map映射，替代多if判断
http {
    map $http_user_agent $device_type {
        default "pc";
        ~* "mobile|android|iphone" "mobile";
        ~* "ipad|tablet" "pad";
    }

    map $host $backend_server {
        default http://default_backend;
        "www.example.com" http://web_backend;
        "api.example.com" http://api_backend;
        "admin.example.com" http://admin_backend;
    }

    server {
        listen 80;
        server_name _;
        # 直接使用map映射的变量，无if陷阱
        proxy_pass $backend_server;
    }
}
```

#### 6.3 用 try_files 指令替代文件存在性判断

`try_files` 指令是 Nginx 原生的文件存在性检查与 URI fallback 指令，专门替代`if (!-f $request_filename)`的伪静态配置，性能更高、行为更稳定，完全规避 if 陷阱。

|错误 if 配置|正确 try_files 替代方案|
|---|---|
|`nginx location / { root /var/www/html; if (!-f $request_filename) { rewrite ^/(.*)$ /index.php?$1 last; } }`|`nginx location / { root /var/www/html; try_files $uri $uri/ /index.php?$query_string; }`|

#### 6.4 用 error_page 指令替代状态码判断

`error_page` 指令是 Nginx 原生的响应状态码处理指令，专门替代基于`$status`的 if 判断，符合 HTTP 规范，无执行逻辑异常。

|错误 if 配置|正确 error_page 替代方案|
|---|---|
|`nginx location / { if ($status = 404) { rewrite ^ /404.html last; } }`|`nginx location / { root /var/www/html; error_page 404 /404.html; }`|

---

### 7. 配置合规性校验与最佳实践

1. **优先替代原则**：所有分支逻辑优先使用`location`、`map`、`try_files`、`error_page`等原生指令实现，仅在无替代方案时使用`if`指令。
2. **作用域约束原则**：优先在`server`块内使用`if`指令；若必须在`location`块内使用，`if`块中仅可包含 rewrite 模块原生指令，禁止使用任何非 rewrite 模块的内容处理指令。
3. **语法禁忌原则**：禁止使用嵌套`if`、多条件复合判断、数值比较运算符；正则包含特殊字符时必须用引号包裹。
4. **测试校验原则**：所有`if`配置修改后，必须先执行`nginx -t`完成语法校验，再通过`curl`工具覆盖条件成立 / 不成立的全场景测试，验证响应头、跳转逻辑、后端服务接收参数是否符合预期。
5. **301 重定向约束原则**：`if`块内的永久重定向规则，必须先使用 302 临时重定向完成全量业务验证，确认无异常后再切换为 301 永久重定向，避免客户端永久缓存错误规则。
6. **性能优化原则**：避免在高频请求的 location 块内使用复杂正则匹配的`if`指令，正则匹配会增加请求处理耗时，优先使用 location 前缀匹配替代。