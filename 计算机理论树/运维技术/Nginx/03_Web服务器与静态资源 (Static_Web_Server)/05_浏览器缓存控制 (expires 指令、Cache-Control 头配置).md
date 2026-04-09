
浏览器缓存是**提升客户端二次访问速度、降低服务器请求量**的核心优化手段，Nginx 通过`expires`指令与`add_header`指令配置`Cache-Control`响应头，控制浏览器对静态资源的缓存策略，让客户端在缓存有效期内直接从本地读取资源，无需向服务器发起请求。

### 3.5.1 缓存的核心原理

当客户端首次请求静态资源时，Nginx 返回资源的同时，在响应头中添加**缓存控制字段**（`Expires`/`Cache-Control`）；客户端接收到资源后，将资源与缓存字段一起缓存至本地；当客户端二次请求该资源时，先检查本地缓存的有效期，若未过期则直接使用本地资源（状态码 304 Not Modified），若已过期则向服务器发起请求验证资源是否更新。

### 3.5.2 expires 指令（核心缓存配置）

`expires`指令是 Nginx 配置浏览器缓存的**简化指令**，可直接设置资源的缓存有效期，Nginx 会自动生成对应的`Expires`与`Cache-Control`响应头，无需手动配置，语法灵活且适配多种场景nginx。

#### 语法与作用域

- 语法：`expires [time] | epoch | max | off;`
- 作用域：`http`、`server`、`location`块，支持按资源类型精细化配置。

#### 核心参数说明

|参数|作用|示例|
|---|---|---|
|time|设置缓存有效期，支持 s（秒）、m（分）、h（时）、d（天）、w（周）|expires 7d;（缓存 7 天）、expires 12h;（缓存 12 小时）|
|modified + time|基于文件**最后修改时间**计算缓存有效期，而非当前时间|expires modified + 1d;（文件修改后缓存 1 天）|
|epoch|设置缓存有效期为 Unix 时间戳 0（1970-01-01），即禁止浏览器缓存|expires epoch;|
|max|设置缓存有效期为 10 年，适配永久不更新的静态资源|expires max;|
|off|关闭 expires 指令的作用，不生成相关缓存响应头|expires off;|

### 3.5.3 Cache-Control 响应头配置（add_header 指令）

`Cache-Control`是 HTTP/1.1 的**标准缓存控制头**，比`Expires`更灵活、优先级更高（`Expires`为 HTTP/1.0 标准，基于绝对时间，易受客户端时间偏差影响）。Nginx 通过`add_header`指令手动配置`Cache-Control`，实现更精细化的缓存控制nginx。

#### 核心`Cache-Control`指令值

|指令值|作用|适用场景|
|---|---|---|
|public|允许所有缓存（浏览器、代理服务器）缓存该资源|公开的静态资源（图片、CSS、JS）|
|private|仅允许浏览器缓存，禁止代理服务器缓存|包含用户信息的资源|
|no-cache|不直接使用本地缓存，需向服务器验证资源是否更新|经常更新的静态资源（如首页 HTML）|
|no-store|禁止所有缓存，每次请求均从服务器获取最新资源|敏感资源（如支付页面、验证码）|
|max-age = 秒|设置缓存的最大有效时间，从请求成功开始计算|通用静态资源，如 max-age=604800（7 天）|

#### 配置示例

```nginx
# 配置public缓存，有效期7天
add_header Cache-Control "public, max-age=604800";
# 配置no-cache，需服务器验证
add_header Cache-Control "no-cache";
# 配置no-store，禁止所有缓存
add_header Cache-Control "no-store";
```

#### 关键特性

- `add_header`默认仅对**200、201、204、301、302**等成功状态码生效，添加`always`参数后对所有状态码生效：`add_header Cache-Control "public" always;`nginx；
- 若当前块配置了`add_header`，则不会继承父块的`add_header`配置，需通过`add_header_inherit merge`实现继承。

### 3.5.4 生产环境精细化缓存配置示例

根据**资源更新频率**对不同类型的静态资源配置差异化的缓存策略，是生产环境的最佳实践，示例如下：

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/static;

    # 图片、视频、音频：更新频率极低，缓存30天
    location ~* \.(jpg|jpeg|png|gif|mp4|mp3|avi)$ {
        expires 30d;
        add_header Cache-Control "public, max-age=2592000";
    }

    # CSS、JS：更新频率低，缓存7天
    location ~* \.(css|js|svg)$ {
        expires 7d;
        add_header Cache-Control "public, max-age=604800";
    }

    # HTML：更新频率高，禁止本地缓存，需服务器验证
    location ~* \.html$ {
        expires epoch;
        add_header Cache-Control "no-cache, max-age=0";
    }

    # 配置文件、日志：敏感资源，禁止所有缓存
    location ~* \.(conf|log|txt)$ {
        expires off;
        add_header Cache-Control "no-store";
        deny all; # 同时禁止外部访问
    }
}
```

### 3.5.5 缓存配置注意事项

1. **缓存失效策略**：对已配置缓存的静态资源，更新后需通过**资源重命名**（如`app.v2.css`）或**添加版本号**（如`app.css?v=2`）让客户端重新请求最新资源，避免客户端使用旧缓存；
2. **避免对动态资源配置缓存**：如 PHP、Java 接口的响应，配置缓存会导致客户端获取到旧的动态数据；
3. **配合 CDN 缓存**：Nginx 的缓存配置会透传至 CDN，需与 CDN 的缓存策略保持一致，避免缓存冲突；
4. **304 状态码优化**：开启`etag`与`last-modified`指令（Nginx 默认开启），让服务器通过资源标识验证资源是否更新，减少 304 请求的资源传输量。

## 本章小结

本章围绕 Nginx 作为 Web 服务器的核心能力，详细讲解了静态资源服务的全量优化配置：首先明确了`root`与`alias`的路径映射逻辑与适用场景，解决了静态资源路径解析的核心问题；其次讲解了首页与目录浏览的配置，适配静态站点的基础访问需求；随后深入解析了 Location 匹配规则的**5 类模式**与**严格的优先级顺序**，这是实现精准路由的核心，也是 Nginx 的必考知识点；接着介绍了 Gzip 压缩优化的核心配置与 CPU 开销解决方案，实现了资源传输体积的大幅减小；最后讲解了浏览器缓存控制的`expires`与`Cache-Control`配置，通过差异化缓存策略提升客户端访问速度、降低服务器压力。

本章内容是 Nginx 静态资源服务的**核心基础**，所有配置均直接影响静态资源的分发效率与用户体验，需重点掌握 Location 匹配优先级、Gzip 压缩的最佳配置、浏览器缓存的精细化策略，同时理解`root`与`alias`的本质区别，避免配置错误导致的 404/403 错误。后续反向代理、负载均衡等进阶功能，均会基于本章的 Location 匹配、路径映射等规则展开。