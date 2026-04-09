
Nginx 的日志体系是服务行为审计、请求溯源、故障定位、性能分析的核心数据依据，监控体系是保障服务高可用、容量规划、异常告警的核心技术支撑。本章严格遵循 Nginx 官方规范，覆盖从基础日志配置、结构化日志格式化、自动化日志轮转，到基础状态监控、企业级 Prometheus 全栈监控体系的完整链路，实现「可观测、可追溯、可预警」的全生命周期管理，是企业级 Nginx 服务运维的核心必备能力。

本章所有核心指令均来自 Nginx 标准内置模块，除特殊说明的第三方监控组件外，无需额外源码编译配置，所有配置均符合 Linux 运维行业标准与可观测性最佳实践。

---

Nginx 的日志体系分为两大核心类型：**访问日志（access_log）** 与 **错误日志（error_log）**，二者分别记录客户端请求的全链路信息与服务运行的异常事件，模块归属、执行逻辑、应用场景有本质差异。

### 1.1 错误日志 error_log

错误日志是 Nginx 故障排查的第一优先级数据源，隶属于**ngx_core_module**核心模块，无需额外编译启用，用于记录 Nginx 服务启动、重载、运行过程中的启动失败、配置错误、资源耗尽、请求处理异常、内核告警等全类型事件，是定位服务不可用、请求异常的核心依据。

#### 1.1.1 官方标准语法

```nginx
error_log file [level];
```

**合法作用域**：`main`、`http`、`server`、`location`

**默认配置**：`error_log logs/error.log error;`

#### 1.1.2 核心参数解析

1. **file 参数**：日志文件的存储路径，支持绝对路径与相对路径（相对 Nginx 编译时指定的 prefix 目录）；特殊值`/dev/null`可丢弃所有错误日志（生产环境不推荐）；支持输出到系统日志`syslog:`。
2. **level 参数**：日志级别，按严重程度从低到高排序，仅记录大于等于指定级别的日志，级别越高，日志输出量越少。合法级别如下：

| 日志级别     | 核心适用场景                                     | 生产环境推荐度    |
| -------- | ------------------------------------------ | ---------- |
| `debug`  | 开发调试场景，输出最详细的底层执行日志，需编译时开启`--with-debug`参数 | 禁止生产环境使用   |
| `info`   | 常规运行信息通知，无异常事件                             | 不推荐生产环境使用  |
| `notice` | 需关注的常规事件，无业务影响                             | 非核心场景可选    |
| `warn`   | 警告事件，服务可正常运行，但存在潜在风险（如配置不规范、连接超时）          | 生产环境推荐默认级别 |
| `error`  | 错误事件，请求处理失败、资源调用异常，业务受影响                   | 生产环境通用默认级别 |
| `crit`   | 严重错误，核心配置失效、资源耗尽，服务部分功能不可用                 | 极简日志场景可选   |
| `alert`  | 告警事件，需立即处理，服务面临整体不可用风险                     | 高安全场景可选    |
| `emerg`  | 紧急崩溃事件，服务完全不可用，无法正常启动                      | 仅极端极简场景使用  |

#### 1.1.3 标准配置示例

```nginx
# main全局块：全局错误日志配置，所有子作用域默认继承
error_log /var/log/nginx/error.log warn;

http {
    # 特定server块：单独配置错误日志，覆盖全局配置
    server {
        listen 443 ssl http2;
        server_name api.example.com;
        # API服务单独记录error日志，级别为error
        error_log /var/log/nginx/api_error.log error;
    }

    server {
        listen 443 ssl http2;
        server_name admin.example.com;
        # 管理后台单独记录warn级别日志，便于权限异常排查
        error_log /var/log/nginx/admin_error.log warn;

        # 特定location：单独配置错误日志
        location /api/upload/ {
            error_log /var/log/nginx/upload_error.log debug;
            client_max_body_size 100m;
            proxy_pass http://upload_backend;
        }
    }
}
```

#### 1.1.4 关键注意事项

1. **不可关闭特性**：Nginx 错误日志无法完全关闭，即使配置`error_log off;`，Nginx 仍会将错误日志输出到名为`off`的文件中；如需丢弃日志，需配置`error_log /dev/null emerg;`，仅保留最紧急的崩溃日志。
2. **作用域继承规则**：子作用域未配置`error_log`时，完全继承父作用域的配置；子作用域一旦配置，会完全覆盖父作用域的日志路径与级别，不再继承。
3. **权限约束**：日志文件路径必须对 Nginx 运行用户有读写权限，父目录必须有执行权限，否则会导致日志写入失败，服务启动异常。
4. **debug 级别限制**：`debug`级别仅在 Nginx 源码编译时添加`--with-debug`参数后可用，生产环境禁止开启，会产生海量日志，严重影响服务性能。

### 1.2 访问日志 access_log

访问日志用于记录所有客户端发送的 HTTP 请求的完整明细信息，隶属于**ngx_http_log_module**标准模块，无需额外编译启用。其核心价值在于用户行为审计、业务统计分析、攻击溯源、性能瓶颈定位、合规审计，是 Nginx 可观测体系的核心数据源。

#### 1.2.1 官方标准语法

```nginx
access_log path [format [buffer=size] [gzip[=level]] [flush=time] [if=condition]];
access_log off;
```

**合法作用域**：`http`、`server`、`location`、`if in location`、`limit_except`

**默认配置**：`access_log logs/access.log combined;`

#### 1.2.2 核心参数解析

|参数|功能说明与生产环境最佳实践|
|---|---|
|`path`|日志文件存储路径，支持绝对路径、相对路径，也支持输出到`syslog:`、内存管道`memory:`|
|`format`|日志格式名称，引用`log_format`指令定义的格式，默认使用 Nginx 内置的`combined`通用格式|
|`buffer=size`|日志内存缓冲区大小，高并发场景下开启，减少磁盘 IO 次数，推荐值`64k`，缓冲区写满后一次性写入磁盘|
|`flush=time`|缓冲区日志的最长缓存时间，即使缓冲区未写满，达到该时间后也会强制写入磁盘，推荐值`5s`，平衡日志实时性与 IO 性能|
|`gzip[=level]`|开启日志压缩，压缩级别 1-9，级别越高压缩率越高，CPU 开销越大，仅适用于日志归档场景，生产环境实时日志不推荐使用|
|`if=condition`|条件日志，当 condition 表达式为非空、非 0 时，才记录该请求的日志，用于过滤无效请求|
|`off`|关闭当前作用域的访问日志，不再记录任何请求信息|

#### 1.2.3 标准配置示例

```nginx
http {
    # 全局访问日志配置，使用combined格式，开启缓冲区优化
    access_log /var/log/nginx/access.log combined buffer=64k flush=5s;
    # 全局错误日志配置
    error_log /var/log/nginx/error.log warn;

    server {
        listen 443 ssl http2;
        server_name www.example.com;
        root /var/www/html;

        # 静态资源：关闭访问日志，减少磁盘IO开销
        location ~* \.(jpg|jpeg|png|gif|webp|css|js|ico|woff2)$ {
            access_log off;
            expires 30d;
        }

        # 健康检查请求：过滤日志，不记录
        location /health {
            access_log off;
            return 200 '{"status":"ok"}';
            add_header Content-Type application/json;
        }

        # 管理后台：单独记录访问日志，使用自定义格式
        location /admin/ {
            access_log /var/log/nginx/admin_access.log admin_format buffer=64k flush=5s;
            proxy_pass http://admin_backend;
        }

        # 兜底路由：继承全局日志配置
        location / {
            index index.html;
        }
    }

    # API服务：单独配置访问日志，开启条件日志，仅记录非200响应
    server {
        listen 443 ssl http2;
        server_name api.example.com;

        # 条件日志：仅当响应状态码非200时记录日志
        map $status $loggable {
            200 0;
            default 1;
        }
        access_log /var/log/nginx/api_error_access.log api_format if=$loggable;

        location / {
            proxy_pass http://api_backend;
        }
    }
}
```

#### 1.2.4 关键注意事项

1. **作用域继承规则**：子作用域未配置`access_log`时，继承父作用域的配置；子作用域配置`access_log off`后，完全关闭当前作用域的日志，不再继承父作用域配置。
2. **高并发 IO 优化**：生产环境高流量场景必须开启`buffer`与`flush`参数，将频繁的小 IO 合并为批量写入，大幅降低磁盘 IO 压力，避免日志写入成为性能瓶颈。
3. **无效日志过滤**：静态资源、健康检查、爬虫探针等无效请求，必须通过`access_log off`或条件日志关闭记录，避免日志文件快速膨胀，浪费磁盘空间与 IO 资源。
4. **日志权限约束**：日志目录必须对 Nginx 运行用户有读写权限，开启缓冲区后，Nginx 重载时会强制将缓冲区内容写入磁盘，需保证路径权限正常。
5. **长连接日志特性**：HTTP 长连接场景下，请求日志仅在请求处理完成后写入，而非连接建立时写入，一个 TCP 长连接内的多个请求会分别记录独立的日志条目。

---
