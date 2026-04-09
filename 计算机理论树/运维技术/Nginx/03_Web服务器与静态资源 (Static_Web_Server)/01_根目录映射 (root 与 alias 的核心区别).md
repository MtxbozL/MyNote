
Nginx 作为高性能 Web 服务器，对静态资源（HTML、CSS、JS、图片、视频等）的处理能力是其核心优势之一，相较于传统 Web 服务器，具备低资源占用、高并发承载、丰富的优化配置等特性。本章围绕 Nginx 静态资源服务的核心配置展开，涵盖路径映射、首页与目录浏览、Location 匹配规则、Gzip 压缩优化、浏览器缓存控制五大核心模块，是实现静态资源高效分发的基础。

---

`root`与`alias`是 Nginx 配置中用于**将请求 URI 映射至服务器本地文件系统路径**的核心指令，二者均用于静态资源的路径解析，但路径拼接逻辑、适用场景、配置规则存在本质区别，配置错误会直接导致 404 资源未找到错误。

### 3.1.1 root 指令

#### 核心逻辑

将**请求的完整 URI**直接拼接到`root`指定的目录路径后，形成服务器本地的完整文件路径，即**本地文件路径 = root 路径 + 请求 URI**。

#### 语法与作用域

- 语法：`root 本地目录路径;`
- 作用域：`http`、`server`、`location`块，支持多层覆盖（location 块优先级最高）。

#### 示例与解析

```nginx
location /static/ {
    root /var/www/myapp; # 无末尾斜杠不影响解析
}
```

当客户端请求`http://example.com/static/image/logo.jpg`时，Nginx 拼接路径为：`/var/www/myapp + /static/image/logo.jpg = /var/www/myapp/static/image/logo.jpg`，并读取该文件返回。

#### 适用场景

URI 路径与服务器本地文件系统的目录结构**完全一致**时，是静态资源根目录映射的默认选择，配置简洁且易维护。

### 3.1.2 alias 指令

#### 核心逻辑

用`alias`指定的本地目录路径，**直接替换 location 块匹配的 URI 前缀部分**，剩余 URI 拼接至替换后的路径后形成本地文件路径，即**本地文件路径 = alias 路径 + （请求 URI - location 匹配前缀）**。

#### 语法与作用域

- 语法：`alias 本地目录路径;`
- 作用域：**仅限 location 块**，是区别于`root`的核心特性之一。

#### 示例与解析

```nginx
location /assets/ {
    alias /var/www/myapp/public/; # 建议末尾加/，避免路径拼接错误
}
```

当客户端请求`http://example.com/assets/css/index.css`时，Nginx 先将`/assets/`替换为`/var/www/myapp/public/`，再拼接剩余 URI`css/index.css`，最终路径为：`/var/www/myapp/public/css/index.css`。

#### 适用场景

需要将**URI 中的某部分路径映射至服务器本地不同目录**时，如前端工程打包后的静态资源目录为`public`，但希望通过`/assets/`路径访问，是实现 URI 与本地路径解耦的关键指令。

### 3.1.3 root 与 alias 核心对比

|对比维度|root 指令|alias 指令|
|---|---|---|
|路径拼接逻辑|本地路径 = root 路径 + 完整请求 URI|本地路径 = alias 路径 + （请求 URI - location 匹配前缀）|
|末尾斜杠要求|无强制要求，Nginx 自动处理|建议以`/`结尾，否则易出现路径拼接错误|
|作用域|http、server、location 块|仅限 location 块|
|正则 location 支持|支持，但路径拼接需谨慎|支持，可通过`$1`/`$2`引用正则捕获组|
|配置简洁性|高，适合路径一致场景|稍复杂，适合路径解耦场景|

### 3.1.4 配置注意事项

1. 确保 Nginx 运行用户对`root`/`alias`指定的本地目录及文件拥有**读权限**，否则会返回 403 权限拒绝错误；
2. `alias`配置正则 location 时，可通过捕获组实现动态路径映射，示例：`location ~ ^/file/(\d+)/ { alias /data/files/$1/; }`；
3. 同一 location 块中不可同时配置`root`与`alias`，否则会导致配置语法错误。
