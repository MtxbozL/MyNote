# 第八章 缓存机制与性能调优

本章聚焦 Nginx 性能优化的核心体系，分为两大核心模块：**代理缓存机制**与**全栈性能调优**。代理缓存通过将后端响应内容本地化存储，大幅降低源站计算与带宽压力，缩短用户访问延迟；全栈性能调优从 Nginx 进程层、系统调用层、Linux 内核层逐级优化，充分释放硬件算力，突破高并发场景下的性能瓶颈，是企业级 Nginx 服务实现百万级并发、毫秒级响应的核心技术支撑，也是 Nginx 学习体系中的「性能天花板」核心内容。

本章所有核心指令均来自 Nginx 标准内置模块，除特殊说明的第三方缓存清除模块外，无需额外源码编译配置，所有配置均遵循 Nginx 官方规范与 Linux 内核技术标准。

---

## 01 代理缓存配置

Nginx 代理缓存能力由**ngx_http_proxy_module**标准模块原生提供，无需额外编译启用。其核心原理是：将反向代理场景下后端源站返回的可缓存 HTTP 响应，持久化存储到服务器本地磁盘 / 内存中，后续针对相同资源的客户端请求，直接从本地缓存返回响应，无需再转发至后端源站处理，实现「一次源站请求，多次缓存响应」，大幅降低源站负载、减少网络往返时延、提升服务并发承载能力。

### 1.1 核心生命周期与缓存判定逻辑

Nginx 代理缓存的完整执行流程严格遵循 HTTP/1.1 缓存规范（RFC 7234），核心判定逻辑如下：

1. 客户端请求到达 Nginx，Nginx 基于`proxy_cache_key`生成唯一缓存键；
2. 以缓存键为索引，查询本地缓存存储中是否存在匹配的、未过期的有效缓存内容；
3. **缓存命中（HIT）**：存在有效缓存，直接向客户端返回缓存内容，请求终止，不转发至源站；
4. **缓存未命中（MISS）**：无有效缓存，将请求转发至后端源站，接收源站响应后，根据缓存规则判定是否可缓存；
5. 可缓存的响应内容写入本地缓存存储，同时返回给客户端；不可缓存的内容直接返回客户端，不写入缓存。

### 1.2 核心指令与语法规范

#### 1.2.1 proxy_cache_path：缓存全局定义指令

该指令是代理缓存的基础配置，**仅可在 http 全局块中配置**，用于定义缓存的存储路径、内存索引、生命周期、容量上限等核心参数，是启用缓存的前置条件。

**官方标准语法**：

nginx

```
proxy_cache_path path 
    [levels=levels] 
    [use_temp_path=on|off]
    keys_zone=name:size 
    [inactive=time] 
    [max_size=size] 
    [manager_files=number] 
    [manager_sleep=time] 
    [manager_threshold=time]
    [loader_files=number] 
    [loader_sleep=time] 
    [loader_threshold=time]
    [purger=on|off] 
    [purger_files=number] 
    [purger_sleep=time] 
    [purger_threshold=time];
```

**核心参数深度解析**：

表格

|参数|合法取值|核心功能与学术原理|
|---|---|---|
|`path`|绝对路径字符串|缓存文件的根存储路径，建议使用独立的高速磁盘分区（SSD 优先），避免与系统盘、日志盘共用，减少 IO 竞争|
|`levels`|1-3 级目录结构，如`1:2`、`2:2:2`|定义缓存文件的目录层级，采用 16 进制哈希目录拆分。Nginx 将缓存键哈希为 16 进制字符串，按层级拆分目录，避免单目录下文件数量过多导致的文件系统寻址性能下降。示例：`levels=1:2`表示两级目录，第一级取哈希值最后 1 位，第二级取倒数 2-3 位|
|`use_temp_path`|`on`/`off`，默认`on`|定义缓存文件临时写入路径。**生产环境必须设置为`off`**，关闭后缓存文件直接写入`path`指定的缓存目录，避免跨文件系统的`rename`系统调用开销，同时消除临时目录与缓存目录不在同一文件系统时的 IO 性能损耗|
|`keys_zone=name:size`|名称自定义，大小如`10m`|定义共享内存索引区的名称与容量。该内存区用于存储所有缓存键的哈希索引、缓存元数据（有效期、命中次数、大小等），所有 Worker 进程共享访问。**容量换算标准**：1MB 共享内存可存储约 8000 条缓存索引记录，10MB 可支持约 8 万条缓存条目，需根据缓存容量合理设置|
|`inactive=time`|时间单位，如`7d`、`1h`，默认`10s`|缓存内容的非活动过期时间。若缓存在该时间窗口内未被任何请求命中，无论其`Cache-Control`设置的有效期是否到期，都会被缓存管理进程从磁盘中清除。该参数是缓存淘汰的核心触发条件，优先级高于源站设置的缓存有效期|
|`max_size=size`|容量单位，如`100g`、`20g`|缓存目录的最大磁盘容量上限。当缓存占用超过该阈值时，Nginx 缓存管理进程会基于 LRU（最近最少使用）算法，淘汰最少被访问的缓存内容，释放磁盘空间|

#### 1.2.2 proxy_cache：缓存启用指令

该指令用于在`server`、`location`块中启用代理缓存，引用`proxy_cache_path`中定义的共享内存索引区名称，实现不同业务场景的差异化缓存策略。

**官方语法**：`proxy_cache zone_name | off;`

**合法作用域**：`http`、`server`、`location`

**默认值**：`off`

> 核心特性：子作用域的配置会完全覆盖父作用域，可在全局启用缓存后，在特定 location 中通过`proxy_cache off`关闭缓存。

#### 1.2.3 proxy_cache_key：缓存键定义指令

该指令用于定义生成缓存唯一索引的键值规则，是缓存命中的核心依据，键值的微小差异会生成完全独立的缓存条目。

**官方语法**：`proxy_cache_key string;`

**合法作用域**：`http`、`server`、`location`

**默认值**：`$scheme$proxy_host$request_uri;`

> 核心原理：默认缓存键由「请求协议 + 代理主机名 + 完整请求 URI」组成，可保证相同资源的请求生成唯一缓存键。自定义时需避免引入高频变化的变量（如`$remote_addr`、`$cookie_uid`），否则会导致缓存完全失效，每个请求都生成独立缓存条目。

#### 1.2.4 缓存有效性控制指令

表格

|指令|官方语法|核心功能|||
|---|---|---|---|---|
|`proxy_cache_valid`|`proxy_cache_valid [code ...] time;`|基于 HTTP 响应状态码设置缓存有效期，可针对不同状态码设置差异化时长。示例：`proxy_cache_valid 200 304 1h;`表示 200、304 响应缓存 1 小时；`proxy_cache_valid any 1m;`表示所有其他状态码缓存 1 分钟|||
|`proxy_cache_min_uses`|`proxy_cache_min_uses number;`|设置请求被缓存的最小命中次数，默认值 1。示例：设置为 3 时，同一请求前 3 次访问均会转发至源站，第 4 次访问才会将响应写入缓存，避免非热点内容占用缓存资源|||
|`proxy_cache_methods`|`proxy_cache_methods GET|HEAD|POST ...;`|设置允许缓存的 HTTP 请求方法，默认仅支持`GET`、`HEAD`方法。如需缓存 POST 请求，需显式添加，注意 POST 请求的缓存键必须包含请求体相关变量，否则会出现缓存错乱|

#### 1.2.5 缓存绕过与不缓存指令

该组指令用于实现精细化的缓存例外控制，核心分为`proxy_cache_bypass`（绕过缓存）与`proxy_no_cache`（不存储缓存），二者的行为边界有本质差异，是生产环境最易出现配置错误的场景。

表格

|指令|官方语法|核心执行逻辑|适用场景|
|---|---|---|---|
|`proxy_cache_bypass`|`proxy_cache_bypass string ...;`|当字符串值为非空、非 0 时，Nginx 直接绕过缓存，将请求转发至源站，**但源站返回的响应仍会正常写入缓存**，更新原有缓存内容|后台预览、强制刷新、灰度发布场景，需获取源站最新内容，同时更新缓存|
|`proxy_no_cache`|`proxy_no_cache string ...;`|当字符串值为非空、非 0 时，Nginx 不从缓存读取内容，同时**源站返回的响应完全不会写入缓存**|后台管理接口、用户中心、带登录态的动态请求、支付接口等不可缓存的场景|

### 1.3 标准生产环境配置示例

#### 1.3.1 全局缓存基础配置

nginx

```
http {
    # 全局代理缓存定义：SSD高速磁盘存储
    proxy_cache_path /data/nginx/cache
        levels=1:2                     # 两级哈希目录，避免单目录文件过多
        use_temp_path=off              # 直接写入缓存目录，消除跨文件系统IO开销
        keys_zone=WEB_CACHE:100m       # 100MB共享内存，可存储约80万条缓存索引
        inactive=7d                     # 7天未命中的缓存自动清除
        max_size=200g;                  # 最大磁盘占用200GB，超出后LRU淘汰

    # 全局缓存默认规则
    proxy_cache_key $scheme$host$request_uri;  # 自定义缓存键，包含访问域名
    proxy_cache_min_uses 3;                     # 非热点内容3次访问后才缓存
    proxy_cache_valid 200 304 12h;             # 正常响应缓存12小时
    proxy_cache_valid 301 302 1h;               # 重定向响应缓存1小时
    proxy_cache_valid any 1m;                    # 其他异常响应缓存1分钟

    # 缓存状态响应头，便于调试与监控
    add_header X-Cache-Status $upstream_cache_status always;
    add_header X-Cache-Key $proxy_cache_key always;

    # 业务server块
    server {
        listen 443 ssl http2;
        server_name www.example.com;
        ssl_certificate /etc/nginx/ssl/example.com.crt;
        ssl_certificate_key /etc/nginx/ssl/example.com.key;

        # 静态资源：强缓存策略
        location ~* \.(jpg|jpeg|png|gif|webp|css|js|ico|woff2)$ {
            proxy_cache WEB_CACHE;
            proxy_cache_valid 200 304 30d;  # 静态资源缓存30天
            proxy_cache_min_uses 1;          # 首次访问即缓存
            proxy_cache_lock on;              # 缓存未命中时，仅放行1个请求到源站，避免惊群效应
            proxy_pass http://static_backend;
            expires 30d;
        }

        # 首页与列表页：中等缓存策略
        location ~* ^/(index|list).*\.html$ {
            proxy_cache WEB_CACHE;
            proxy_cache_valid 200 304 1h;
            proxy_cache_lock on;
            # 带预览参数时绕过缓存，同时更新缓存
            proxy_cache_bypass $arg_preview;
            proxy_pass http://web_backend;
            expires 1h;
        }

        # 动态接口与管理后台：完全关闭缓存
        location ~* ^/(api|admin|user|pay)/ {
            proxy_cache off;                  # 关闭缓存
            proxy_no_cache 1;                 # 强制不存储缓存
            proxy_cache_bypass 1;             # 强制绕过缓存
            proxy_pass http://api_backend;
            # 禁用客户端缓存
            add_header Cache-Control "no-store, no-cache, must-revalidate, max-age=0" always;
        }

        # 兜底路由
        location / {
            proxy_cache WEB_CACHE;
            proxy_pass http://web_backend;
        }
    }
}
```

### 1.4 关键注意事项与常见陷阱

1. **缓存键唯一性约束**：禁止在`proxy_cache_key`中引入客户端唯一变量（如`$remote_addr`、`$cookie_sessionid`），否则会导致缓存命中率趋近于 0，完全失去缓存价值。
2. **缓存目录 IO 性能**：缓存目录必须部署在 SSD 高速磁盘上，机械硬盘的随机 IO 性能无法支撑高并发缓存读写场景，会成为性能瓶颈。
3. **HTTP 缓存规范兼容**：Nginx 默认遵循源站响应的`Cache-Control`、`Expires`、`Pragma`头规则，若源站返回`Cache-Control: no-cache`、`no-store`、`private`，即使配置了`proxy_cache_valid`，也不会缓存该响应。可通过`proxy_ignore_headers`指令忽略源站缓存头，强制缓存，仅推荐静态资源场景使用。
4. **缓存惊群效应规避**：高并发场景下必须开启`proxy_cache_lock on`，当多个请求同时命中同一个未缓存的资源时，仅放行 1 个请求到源站，其余请求等待缓存生成后直接读取，避免源站被突发请求打垮。
5. **缓存命中率监控**：通过`$upstream_cache_status`变量监控缓存状态，核心状态包括`HIT`（命中）、`MISS`（未命中）、`EXPIRED`（过期）、`BYPASS`（绕过），生产环境需保证缓存命中率≥90%，否则需优化缓存规则。

---
