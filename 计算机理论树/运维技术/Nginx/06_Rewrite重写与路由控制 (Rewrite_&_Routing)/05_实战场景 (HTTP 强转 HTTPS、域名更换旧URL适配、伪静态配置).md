
本章实战场景基于前 4 节的语法规则、执行逻辑与避坑指南，覆盖企业级生产环境中 rewrite 体系的 3 大核心高频场景，所有配置均遵循 Nginx 官方最佳实践，完全规避 "If is Evil" 等陷阱，兼顾性能、稳定性与 SEO 合规性。

### 场景 1：HTTP 强转 HTTPS

#### 1.1 需求背景

全站 HTTPS 化是现代 Web 服务的基础安全要求，需将所有 HTTP 协议的请求，永久重定向到 HTTPS 协议，同时完整保留原始请求的域名、URI 路径、查询参数，保证用户访问无感知，且符合 SEO 权重传递规范。

#### 1.2 实现原理

基于 Nginx 多 server 块分离设计，独立监听 80 端口的 HTTP 请求，通过`return 301`指令实现永久重定向，无需正则匹配与`if`判断，性能最优、无任何陷阱；重定向地址通过`$scheme`、`$host`、`$request_uri`内置变量动态生成，完整保留原始请求信息。

#### 1.3 标准最佳配置

```nginx
# 独立HTTP server块，专门处理80端口请求，无多余逻辑
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    # 核心配置：301永久重定向到HTTPS，完整保留原始请求URI与参数
    return 301 https://$host$request_uri;
}

# HTTPS server块，处理443端口的正常业务请求
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com www.example.com;

    # SSL证书配置
    ssl_certificate /etc/nginx/ssl/example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;

    # 业务配置
    root /var/www/example.com;
    index index.html index.php;

    # HSTS配置：强制浏览器长期使用HTTPS访问，避免降级跳转
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
}
```

#### 1.4 关键注意事项

1. **禁止在 HTTPS server 块内使用 if 判断实现跳转**：常见错误配置为在 443 端口的 server 块内通过`if ($scheme = http)`实现跳转，该配置会触发 "If is Evil" 陷阱，且所有 HTTPS 请求都会执行一次无用的 if 判断，性能损耗极大。
2. ** 必须使用保留完整请求信息：uri`仅包含路径部分，会丢失查询参数；`$request_uri` 包含完整的原始请求路径与查询字符串，保证跳转后请求参数无丢失。
3. **配合 HSTS 实现强制 HTTPS**：添加 HSTS 响应头，让浏览器直接在本地发起 HTTPS 请求，无需经过服务端跳转，提升访问速度与安全性。
4. **泛域名适配**：该配置天然支持泛域名与多域名，无需修改即可适配同一 server 块下的所有域名，无需为每个域名单独配置跳转规则。

---

### 场景 2：域名更换与旧 URL 适配

#### 2.1 需求背景

业务域名更换（旧域名`old-example.com`切换为新域名`new-example.com`），或网站 URL 结构重构（旧 URL`/thread-123.html`调整为新 URL`/topic/123.html`），需将所有旧 URL 永久重定向到对应的新 URL，完整保留 SEO 权重、用户书签与搜索引擎收录，同时保证访问无异常。

#### 2.2 实现原理

1. 全域名迁移：通过独立 server 块监听旧域名，使用`return 301`实现全量 URL 的永久重定向，完整保留路径与参数；
2. URL 结构重构：通过`rewrite`正则匹配捕获核心参数，配合`permanent`标志位实现一对一的 URL 映射，保证 SEO 权重传递；
3. 所有重定向均使用 301 永久重定向，符合搜索引擎规范，实现权重的完整转移。

#### 2.3 标准最佳配置

##### 2.3.1 全域名迁移配置

```nginx
# 旧域名专属server块，处理所有旧域名请求
server {
    listen 80;
    listen 443 ssl;
    server_name old-example.com www.old-example.com;

    # 旧域名SSL证书配置（必须配置，否则HTTPS旧域名访问会报证书错误）
    ssl_certificate /etc/nginx/ssl/old-example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/old-example.com.key;

    # 核心配置：全量301重定向到新域名，完整保留原始请求URI与参数
    return 301 https://www.new-example.com$request_uri;
}

# 新域名业务server块
server {
    listen 443 ssl http2;
    server_name www.new-example.com;
    # 业务配置省略
}
```

##### 2.3.2 URL 结构重构适配配置

```nginx
server {
    listen 443 ssl http2;
    server_name www.example.com;
    root /var/www/example.com;

    # 场景1：旧文章URL /article-<id>.html 适配新URL /archives/<id>/
    rewrite ^/article-(\d+)\.html$ /archives/$1/ permanent;

    # 场景2：旧论坛URL /thread-<id>-<page>.html 适配新URL /topic/<id>?page=<page>
    rewrite ^/thread-(\d+)-(\d+)\.html$ /topic/$1?page=$2 permanent;

    # 场景3：旧分类URL /category.php?cid=<id> 适配新URL /category/<id>/
    # 注：query string参数匹配需配合if判断，仅在无替代方案时使用
    if ($query_string ~* "cid=(\d+)") {
        set $cid $1;
        rewrite ^/category.php$ /category/$cid/? permanent;
    }

    # 业务配置省略
}
```

#### 2.4 关键注意事项

1. **旧域名 HTTPS 证书必须保留**：域名更换后，旧域名的 HTTPS 证书必须持续有效，否则用户通过 HTTPS 访问旧域名时，会出现证书安全错误，无法完成跳转。
2. **正则锚定符必须严格使用**：正则表达式必须使用`^`和`$`锚定 URI 的开头与结尾，避免泛匹配导致的错误跳转，例如`/article-123.html`不会被`/article-1234.html`误匹配。
3. **301 重定向必须先测试验证**：上线前先使用 302 临时重定向完成全场景测试，确认跳转路径、参数、状态码完全符合预期后，再切换为 301 永久重定向，避免错误规则被客户端永久缓存。
4. **URL 适配优先使用 location 匹配**：若重构的 URL 有统一前缀，优先使用 location 匹配替代 rewrite 正则，性能更高、无正则开销，例如：

    ```nginx
    # 优于rewrite的location适配方案
    location ^~ /article- {
        rewrite ^/article-(\d+)\.html$ /archives/$1/ permanent;
    }
    ```
    
5. **SEO 补充配置**：在新域名的 robots.txt 中配置允许搜索引擎抓取，同时通过搜索引擎站长平台提交域名变更申请，加速权重转移。

---

### 场景 3：伪静态配置

#### 3.1 需求背景

动态 Web 应用（基于 PHP、Java、Python 等开发）的原生 URL 通常包含`index.php`、查询参数等动态内容，不利于搜索引擎 SEO 收录与用户记忆。伪静态的核心是将动态 URL 重写为语义化的静态 URL 格式（`.html`、`.htm`后缀，无查询参数），同时在服务端内部将重写后的 URI 转发给动态应用处理，客户端无感知。

#### 3.2 实现原理

基于 Nginx 的`try_files`指令（官方推荐方案，完全规避 if 陷阱），先检查请求的 URI 是否对应真实的静态文件（图片、CSS、JS 等），若不存在则将请求转发给应用的入口文件（通常为`index.php`），由应用完成路由解析；复杂场景配合`rewrite`指令实现精准的 URI 重写，保证伪静态规则的兼容性。

#### 3.3 主流框架标准伪静态配置

##### 3.3.1 通用型 PHP CMS 伪静态（WordPress、Typecho 等）

```nginx
server {
    listen 443 ssl http2;
    server_name example.com www.example.com;
    root /var/www/wordpress;
    index index.html index.php;

    # SSL配置省略

    # 核心伪静态配置：官方推荐try_files方案，无if陷阱
    location / {
        # 先检查$uri是否为真实存在的静态文件/目录，不存在则转发给index.php
        try_files $uri $uri/ /index.php?$query_string;
    }

    # PHP解析配置
    location ~ \.php$ {
        fastcgi_pass unix:/run/php-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

##### 3.3.2 Laravel/ThinkPHP 等现代 PHP 框架伪静态

```nginx
server {
    listen 443 ssl http2;
    server_name laravel.example.com;
    # 框架入口目录为public，必须指向该目录
    root /var/www/laravel/public;
    index index.php index.html;

    # SSL配置省略

    # 核心伪静态配置
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # 安全配置：禁止访问敏感文件
    location ~ /\.ht {
        deny all;
    }

    # PHP解析配置
    location ~ \.php$ {
        fastcgi_pass unix:/run/php-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        # 框架PATH_INFO兼容配置
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

##### 3.3.3 自定义精准伪静态规则

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;
    root /var/www/html;
    index index.php;

    # 自定义伪静态规则：语义化URL重写
    # 文章详情页：/article/123.html => /index.php?m=article&a=detail&id=123
    rewrite ^/article/(\d+)\.html$ /index.php?m=article&a=detail&id=$1 last;

    # 分类列表页：/category/tech/2.html => /index.php?m=category&a=list&slug=tech&page=2
    rewrite ^/category/([a-zA-Z0-9_-]+)/(\d+)\.html$ /index.php?m=category&a=list&slug=$1&page=$2 last;

    # 用户主页：/user/zhangsan => /index.php?m=user&a=home&username=zhangsan
    rewrite ^/user/([a-zA-Z0-9_-]+)$ /index.php?m=user&a=home&username=$1 last;

    # 兜底转发规则
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # PHP解析配置省略
}
```

#### 3.4 关键注意事项

1. **优先使用 try_files 替代 if 判断**：传统伪静态配置常用`if (!-f $request_filename)`实现，该配置会触发 "If is Evil" 陷阱，`try_files`是 Nginx 官方为伪静态场景设计的原生指令，性能更高、无任何副作用。
2. **last 标志位的正确使用**：自定义 rewrite 伪静态规则必须使用`last`标志位，重写后的 URI 需要匹配到 PHP 解析的 location 块，若使用`break`会导致 PHP 解析失效，直接返回 404 错误。
3. **静态资源排除**：`try_files`指令会自动排除真实存在的静态文件（图片、CSS、JS、字体等），无需额外配置 rewrite 排除规则，保证静态资源的正常访问。
4. **正则性能优化**：自定义 rewrite 规则避免使用过于复杂的正则表达式，优先使用前缀匹配的 location 块缩小正则匹配范围，减少高频请求的性能损耗。
5. **入口文件安全**：禁止将应用的核心代码目录暴露在 Web 根目录下，现代 PHP 框架必须将 Web 根目录指向`public`目录，仅入口文件`index.php`可被 Web 访问，避免敏感文件泄露。