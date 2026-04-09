
Gzip 是 Nginx 提供的**HTTP 内容压缩功能**，基于 DEFLATE 算法将文本类静态资源（HTML、CSS、JS、JSON 等）压缩后传输至客户端，可大幅减小资源传输体积（通常压缩率达 70% 以上），降低服务器带宽占用，提升客户端资源加载速度。Gzip 的核心是**用少量 CPU 开销换取带宽与访问速度的提升**，是静态资源优化的必选配置。

### 3.4.1 核心配置指令集

Gzip 相关指令均配置在`http`、`server`、`location`块中，核心指令及说明如下，所有指令默认关闭或为保守值，需手动优化配置：

| 指令                | 语法                                              | 核心作用                                                  | 推荐值                                                                                  |
| ----------------- | ----------------------------------------------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------ |
| gzip              | gzip on \| off;                                 | 开启 / 关闭 Gzip 压缩功能                                     | on                                                                                   |
| gzip_min_length   | gzip_min_length 大小；                             | 开启压缩的最小文件大小，小于该值的文件不压缩（避免小文件压缩得不偿失）                   | 1k                                                                                   |
| gzip_comp_level   | gzip_comp_level 1-9;                            | 压缩级别，1 为最低（压缩率低、CPU 消耗小），9 为最高（压缩率高、CPU 消耗大）          | 4-6（平衡压缩率与 CPU）                                                                      |
| gzip_types        | gzip_types 媒体类型...;                             | 指定需要压缩的资源 MIME 类型，默认仅压缩 text/html                     | text/plain text/css application/json application/javascript text/xml application/xml |
| gzip_vary         | gzip_vary on \| off;                            | 开启后添加`Vary: Accept-Encoding`响应头，告诉代理服务器缓存压缩 / 未压缩两个版本 | on                                                                                   |
| gzip_disable      | gzip_disable 正则表达式；                             | 对匹配的客户端（如低版本 IE）禁用 Gzip 压缩                            | "MSIE (1-6)."（禁用 IE6 及以下）                                                            |
| gzip_http_version | gzip_http_version 1.0 \| 1.1;                   | 指定开启 Gzip 的 HTTP 协议版本                                 | 1.1（主流客户端均支持）                                                                        |
| gzip_proxied      | gzip_proxied off \| expired \| no-cache \| ...; | 对反向代理的请求是否开启压缩                                        | any（所有代理请求均压缩）                                                                       |

### 3.4.2 生产环境最佳配置示例

```nginx
http {
    # 开启Gzip压缩
    gzip on;
    # 最小压缩文件大小1k
    gzip_min_length 1k;
    # 压缩级别6，平衡性能与压缩率
    gzip_comp_level 6;
    # 需要压缩的资源类型，覆盖主流文本与前端资源
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript image/svg+xml;
    # 添加Vary响应头，适配代理缓存
    gzip_vary on;
    # 禁用低版本IE的Gzip压缩
    gzip_disable "MSIE (1-6)\.";
    # 仅对HTTP/1.1及以上开启压缩
    gzip_http_version 1.1;
    # 对所有反向代理请求开启压缩
    gzip_proxied any;
}
```

### 3.4.3 解决 CPU 消耗问题

Gzip 压缩会消耗服务器 CPU 资源，尤其是高压缩级别与高并发场景下，需通过以下方式优化，平衡压缩效果与 CPU 开销：

1. **合理设置压缩级别**：避免使用 9 级最高压缩，生产环境推荐 4-6 级，压缩率与 CPU 消耗达到最优平衡；
2. **精准配置压缩类型**：仅对**文本类资源**开启压缩，图片、视频、音频等二进制资源本身已做压缩（如 JPG、PNG、MP4），再次 Gzip 压缩无效果且浪费 CPU；
3. **设置最小压缩文件大小**：通过`gzip_min_length`忽略小文件（如 1k 以下），小文件压缩后的体积减少有限，却会消耗 CPU 资源；
4. **多核 CPU 利用**：结合`worker_processes auto`配置，让 Nginx 充分利用多核 CPU，分散压缩的 CPU 开销；
5. **静态资源预压缩**：对高频访问的静态资源（如前端打包后的 JS/CSS），在部署前通过工具（如 gzip、zopfli）预压缩为`.gz`文件，Nginx 直接返回预压缩文件，避免运行时压缩消耗 CPU。

### 3.4.4 预压缩资源的 Nginx 配置

```nginx
location ~* \.(js|css|html|svg)$ {
    root /var/www/static;
    # 优先返回预压缩的.gz文件
    gzip_static on;
    # 关闭运行时压缩，避免重复压缩
    gzip off;
    expires 7d; # 配合浏览器缓存
}
```

**注意**：`gzip_static`模块默认未编译，源码编译时需添加`--with-http_gzip_static_module`参数。
