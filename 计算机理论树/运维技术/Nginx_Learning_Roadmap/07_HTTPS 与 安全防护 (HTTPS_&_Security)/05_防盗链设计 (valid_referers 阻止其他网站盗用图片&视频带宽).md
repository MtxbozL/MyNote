
## 05 防盗链设计

Nginx 的防盗链能力由**ngx_http_referer_module**标准模块提供，核心原理是通过校验 HTTP 请求头中的`Referer`字段，识别请求的来源站点，阻止非授权站点盗用本站的图片、视频、静态资源等带宽资源，降低服务器带宽成本，防止资源非法分发。

### 5.1 核心指令与语法

#### 5.1.1 valid_referers 指令

`valid_referers`是防盗链的核心指令，用于定义合法的 Referer 来源，生成内置变量`$invalid_referer`：当请求的 Referer 匹配合法规则时，`$invalid_referer`值为空字符串；不匹配时，值为`1`。

**官方语法**：`valid_referers none | blocked | server_names | string ...;`

**合法作用域**：`server`、`location`

#### 5.1.2 合法规则参数详解

表格

|参数|匹配规则|适用场景|
|---|---|---|
|`none`|匹配请求头中无 Referer 字段的请求|允许浏览器直接输入地址访问、本地文件跳转访问|
|`blocked`|匹配 Referer 字段存在但被防火墙 / 代理修改 / 隐藏的请求（无 http/https 协议前缀）|兼容代理、VPN 环境下的正常访问|
|`server_names`|匹配 Referer 字段中的域名与当前 server 块的`server_name`一致的请求|允许本站自身的资源引用|
|字符串|支持域名通配符、正则表达式，匹配指定的合法来源域名|配置合作站点、CDN 节点等授权域名|

### 5.2 执行逻辑与匹配规则

1. Nginx 收到请求后，提取请求头中的`Referer`字段；
2. 按`valid_referers`指令配置的规则顺序，依次匹配 Referer 字段；
3. 匹配到任意一条合法规则后，立即终止匹配，`$invalid_referer`置为空；
4. 所有规则均不匹配时，`$invalid_referer`置为`1`，判定为非法盗链请求。

### 5.3 标准配置示例

#### 5.3.1 基础图片 / 视频防盗链

nginx

```
server {
    listen 443 ssl http2;
    server_name www.example.com;
    root /var/www/html;

    # 针对图片、视频、音频资源配置防盗链
    location ~* \.(jpg|jpeg|png|gif|bmp|webp|mp4|avi|flv|wmv|mp3|flac)$ {
        # 定义合法Referer来源
        valid_referers none blocked server_names
            *.example.com
            *.baidu.com
            *.google.com
            *.bing.com;
        # 非法盗链请求，返回403禁止访问
        if ($invalid_referer) {
            return 403;
        }
        # 缓存配置
        expires 30d;
        access_log off;
    }
}
```

#### 5.3.2 进阶防盗链：返回自定义防盗图

nginx

```
server {
    listen 443 ssl http2;
    server_name www.example.com;
    root /var/www/html;

    location ~* \.(jpg|jpeg|png|gif|webp)$ {
        valid_referers none blocked server_names *.example.com;
        if ($invalid_referer) {
            # 盗链请求重定向到防盗提示图片，永久缓存
            rewrite ^/ https://www.example.com/static/forbid_hotlink.webp permanent;
        }
        expires 30d;
    }
}
```

#### 5.3.3 严格防盗链：仅允许本站引用

nginx

```
server {
    listen 443 ssl http2;
    server_name resource.example.com;
    root /var/www/resource;

    # 仅允许本站引用，禁止无Referer和第三方站点访问
    location ~* \.(zip|tar.gz|exe|iso)$ {
        valid_referers server_names *.example.com;
        if ($invalid_referer) {
            return 403;
        }
        # 限制单IP下载速度
        limit_rate 1m;
    }
}
```

### 5.4 关键注意事项

1. **Referer 伪造风险**：Referer 请求头可被客户端随意伪造，该方案仅能防止普通的盗链行为，无法抵御刻意的伪造 Referer 攻击，高安全场景需配合 URL 签名、Cookie 校验、Token 鉴权等方案。
2. **谨慎禁用 none 参数**：禁用`none`参数会导致浏览器直接输入资源地址无法访问，同时部分搜索引擎快照、移动端分享场景会丢失 Referer，导致正常访问被拦截，生产环境建议保留`none blocked`参数。
3. **CDN 场景适配**：当静态资源使用 CDN 加速时，需将 CDN 的回源节点 IP、域名加入合法 Referer 列表，同时建议将防盗链配置在 CDN 节点，减少源站请求压力。
4. **正则匹配优化**：仅针对需要防盗链的资源类型配置 location 匹配，避免对 HTML、CSS、JS 等页面资源配置，导致页面渲染异常。
5. **日志记录**：可开启盗链请求的日志记录，统计盗链来源站点，针对性封禁恶意站点。

---
