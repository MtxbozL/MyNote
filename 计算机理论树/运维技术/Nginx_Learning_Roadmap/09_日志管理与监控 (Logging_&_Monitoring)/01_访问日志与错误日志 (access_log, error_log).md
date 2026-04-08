# 第九章 日志管理与监控

Nginx 的日志体系是服务行为审计、请求溯源、故障定位、性能分析的核心数据依据，监控体系是保障服务高可用、容量规划、异常告警的核心技术支撑。本章严格遵循 Nginx 官方规范，覆盖从基础日志配置、结构化日志格式化、自动化日志轮转，到基础状态监控、企业级 Prometheus 全栈监控体系的完整链路，实现「可观测、可追溯、可预警」的全生命周期管理，是企业级 Nginx 服务运维的核心必备能力。

本章所有核心指令均来自 Nginx 标准内置模块，除特殊说明的第三方监控组件外，无需额外源码编译配置，所有配置均符合 Linux 运维行业标准与可观测性最佳实践。

---

## 01 访问日志与错误日志

Nginx 的日志体系分为两大核心类型：**访问日志（access_log）** 与 **错误日志（error_log）**，二者分别记录客户端请求的全链路信息与服务运行的异常事件，模块归属、执行逻辑、应用场景有本质差异。

### 1.1 错误日志 error_log

错误日志是 Nginx 故障排查的第一优先级数据源，隶属于**ngx_core_module**核心模块，无需额外编译启用，用于记录 Nginx 服务启动、重载、运行过程中的启动失败、配置错误、资源耗尽、请求处理异常、内核告警等全类型事件，是定位服务不可用、请求异常的核心依据。

#### 1.1.1 官方标准语法

nginx

```
error_log file [level];
```

**合法作用域**：`main`、`http`、`server`、`location`

**默认配置**：`error_log logs/error.log error;`

#### 1.1.2 核心参数解析

1. **file 参数**：日志文件的存储路径，支持绝对路径与相对路径（相对 Nginx 编译时指定的 prefix 目录）；特殊值`/dev/null`可丢弃所有错误日志（生产环境不推荐）；支持输出到系统日志`syslog:`。
2. **level 参数**：日志级别，按严重程度从低到高排序，仅记录大于等于指定级别的日志，级别越高，日志输出量越少。合法级别如下：
    
    表格
    
    |日志级别|核心适用场景|生产环境推荐度|
    |---|---|---|
    |`debug`|开发调试场景，输出最详细的底层执行日志，需编译时开启`--with-debug`参数|禁止生产环境使用|
    |`info`|常规运行信息通知，无异常事件|不推荐生产环境使用|
    |`notice`|需关注的常规事件，无业务影响|非核心场景可选|
    |`warn`|警告事件，服务可正常运行，但存在潜在风险（如配置不规范、连接超时）|生产环境推荐默认级别|
    |`error`|错误事件，请求处理失败、资源调用异常，业务受影响|生产环境通用默认级别|
    |`crit`|严重错误，核心配置失效、资源耗尽，服务部分功能不可用|极简日志场景可选|
    |`alert`|告警事件，需立即处理，服务面临整体不可用风险|高安全场景可选|
    |`emerg`|紧急崩溃事件，服务完全不可用，无法正常启动|仅极端极简场景使用|
    

#### 1.1.3 标准配置示例

nginx

```
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

nginx

```
access_log path [format [buffer=size] [gzip[=level]] [flush=time] [if=condition]];
access_log off;
```

**合法作用域**：`http`、`server`、`location`、`if in location`、`limit_except`

**默认配置**：`access_log logs/access.log combined;`

#### 1.2.2 核心参数解析

表格

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

nginx

```
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

## 02 自定义日志格式 log_format

`log_format` 指令用于定义访问日志的输出格式，隶属于**ngx_http_log_module**标准模块，是实现日志结构化、字段自定义、对接第三方日志分析平台的核心指令。

### 2.1 官方标准语法与约束

nginx

```
log_format name [escape=default|json|none] string ...;
```

**合法作用域**：仅可在`http`全局块中配置，禁止在`server`、`location`等子作用域中定义

**默认配置**：Nginx 内置`combined`通用日志格式，无需手动定义即可使用

> 核心约束：`log_format`必须在 http 全局块的顶层定义，所有 server 块、location 块仅可引用已定义的格式名称，无法在子作用域内新增或修改格式定义。

### 2.2 核心参数解析

1. **name 参数**：日志格式的自定义名称，全局唯一，区分大小写，用于`access_log`指令引用。
2. **escape 参数**：Nginx 1.11.8 版本新增，用于控制日志中特殊字符的转义规则，是结构化日志的核心配置：
    
    表格
    
    |取值|核心规则|适用场景|
    |---|---|---|
    |`default`|默认值，对特殊字符（双引号、反斜杠、换行、制表符）做 URL 编码转义|非结构化文本日志|
    |`json`|按照 JSON 规范对特殊字符做转义，保证 JSON 格式的合法性，避免格式损坏|结构化 JSON 日志，对接 ELK/Filebeat|
    |`none`|关闭所有转义，特殊字符原样输出|自定义转义规则的特殊场景，生产环境不推荐|
    
3. **string 参数**：日志格式的模板字符串，由固定文本与 Nginx 内置变量组成，多个字符串会自动拼接为完整的一行日志，每行日志对应一个客户端请求。

### 2.3 内置默认格式与核心变量

Nginx 内置`combined`通用日志格式，是绝大多数 Web 服务的标准格式，定义如下：

nginx

```
log_format combined '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent"';
```

#### 2.3.1 核心基础变量解析

表格

|变量名|变量含义|
|---|---|
|`$remote_addr`|客户端真实 IP 地址（反向代理场景需配合 real_ip 模块获取）|
|`$remote_user`|HTTP 基础认证的用户名，无认证时为空字符串|
|`$time_local`|服务器本地时间，格式为`[dd/Mon/yyyy:hh:mm:ss +0800]`|
|`$request`|完整的原始请求行，包含请求方法、URI、HTTP 协议版本|
|`$status`|响应 HTTP 状态码，是业务异常、攻击行为的核心判断依据|
|`$body_bytes_sent`|发送给客户端的响应体字节数，不包含响应头，用于带宽统计|
|`$http_referer`|请求的 Referer 头，记录请求来源页面，用于防盗链、溯源分析|
|`$http_user_agent`|客户端 UA 信息，记录浏览器、操作系统、爬虫类型|

#### 2.3.2 进阶性能与业务变量

生产环境自定义格式时，推荐添加以下进阶变量，用于性能分析、全链路追踪、业务审计：

表格

|变量名|变量含义|核心价值|
|---|---|---|
|`$request_time`|完整请求处理耗时，单位秒，精确到毫秒|定位慢请求、性能瓶颈的核心指标|
|`$upstream_response_time`|上游服务响应耗时，单位秒|区分 Nginx 性能问题与后端服务性能问题|
|`$upstream_status`|上游服务返回的响应状态码|定位后端服务异常的核心依据|
|`$request_id`|Nginx 自动生成的唯一请求 ID，16 位随机字符串|全链路追踪，串联 Nginx 日志与后端服务日志|
|`$host`|请求的 Host 头，即访问域名|多域名站点的日志区分与统计|
|`$request_method`|HTTP 请求方法（GET/POST/PUT/DELETE 等）|接口审计、攻击行为识别|
|`$request_uri`|完整的原始请求 URI，包含查询参数|业务请求溯源、攻击 payload 定位|
|`$http_x_forwarded_for`|X-Forwarded-For 头，记录代理链路中的客户端 IP|反向代理、CDN 场景下的真实 IP 溯源|
|`$ssl_protocol`|TLS 协议版本|HTTPS 安全合规审计|
|`$tcpinfo_rtt`|TCP 连接往返时延，单位微秒|客户端网络质量分析|

### 2.4 生产环境标准自定义格式

#### 2.4.1 通用文本日志格式

适用于常规运维场景、本地日志分析，兼顾字段完整性与可读性：

nginx

```
http {
    # 生产环境通用自定义日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" "$http_user_agent" '
                    '"$http_x_forwarded_for" $request_id $request_time $upstream_response_time $upstream_status';

    # 全局引用该格式
    access_log /var/log/nginx/access.log main buffer=64k flush=5s;
}
```

#### 2.4.2 结构化 JSON 日志格式（对接 ELK/Filebeat）

企业级日志分析场景的标准方案，JSON 格式无需正则解析，可被 Filebeat、Logstash、Elasticsearch 直接提取字段，避免解析错误，大幅提升日志处理效率。

nginx

```
http {
    # 企业级JSON日志格式，适配ELK/Filebeat，开启json转义
    log_format json_elk escape=json
        '{'
            '"timestamp":"$time_iso8601",'
            '"request_id":"$request_id",'
            '"client_ip":"$remote_addr",'
            '"remote_user":"$remote_user",'
            '"domain":"$host",'
            '"request_method":"$request_method",'
            '"request_uri":"$request_uri",'
            '"server_protocol":"$server_protocol",'
            '"status":$status,'
            '"upstream_status":"$upstream_status",'
            '"body_bytes_sent":$body_bytes_sent,'
            '"request_time":$request_time,'
            '"upstream_response_time":"$upstream_response_time",'
            '"http_referer":"$http_referer",'
            '"http_user_agent":"$http_user_agent",'
            '"x_forwarded_for":"$http_x_forwarded_for",'
            '"ssl_protocol":"$ssl_protocol",'
            '"tcp_rtt":$tcpinfo_rtt'
        '}';

    # API服务使用JSON格式日志，对接ELK
    server {
        listen 443 ssl http2;
        server_name api.example.com;
        access_log /var/log/nginx/api_access.log json_elk buffer=128k flush=5s;
        location / {
            proxy_pass http://api_backend;
        }
    }
}
```

### 2.5 关键注意事项与最佳实践

1. **转义规则强制约束**：JSON 格式日志必须配置`escape=json`，否则当请求中包含双引号、换行符等特殊字符时，会导致 JSON 格式损坏，无法被日志平台解析。
2. **字段完整性规范**：自定义格式必须包含`$request_id`唯一请求 ID，实现 Nginx 与后端服务的全链路追踪，是微服务架构下故障定位的核心能力。
3. **数值类型规范**：JSON 格式中，数值类型变量（如`$status`、`$request_time`、`$body_bytes_sent`）禁止用双引号包裹，保证 Elasticsearch 可按数值类型索引，支持范围查询与统计。
4. **格式复用原则**：全局仅定义 2-3 种通用格式，避免为每个 server 块单独定义格式，降低运维复杂度；子作用域仅可引用全局定义的格式，禁止重复定义。
5. **敏感信息脱敏**：日志中禁止记录用户密码、Token、身份证号等敏感信息，可通过 Nginx 变量过滤或日志平台脱敏规则处理，满足数据安全合规要求。
6. **时区规范**：推荐使用`$time_iso8601`替代`$time_local`，前者为 ISO8601 标准时间格式，带有时区信息，适配跨地域部署场景，日志平台解析无需额外时区转换。

---

## 03 日志切割 / 归档

Nginx 本身不提供自动日志切割能力，随着服务持续运行，日志文件会持续膨胀，导致磁盘空间耗尽、日志查询与归档效率极低，甚至引发服务 IO 性能瓶颈。Linux 系统原生的**logrotate**工具是行业标准的日志切割方案，无需额外安装，可实现自动化、平滑的日志切割、压缩、归档、清理全流程，不中断 Nginx 服务。

### 3.1 logrotate 核心工作原理

logrotate 是 Linux 系统内置的日志轮转工具，通过系统定时任务`crontab`每日自动执行，核心执行流程如下：

1. 读取`/etc/logrotate.d/`目录下的 Nginx 专属配置文件，匹配目标日志文件；
2. 按照配置的规则，对达到切割条件的日志文件进行重命名归档；
3. 向 Nginx 主进程发送`USR1`信号，触发 Nginx 重新打开新的日志文件，实现平滑切换，无请求丢失、无服务中断；
4. 按照配置规则，对旧日志进行压缩、保留指定份数、过期自动删除。

> 核心关键：`USR1`信号是 Nginx 官方指定的「重新打开日志文件」信号，属于轻量级操作，仅重新加载日志文件句柄，不重载配置、不重启进程，完全不影响业务请求处理，禁止使用`HUP`信号（重载配置）替代。

### 3.2 logrotate 标准配置详解

Nginx 通过 yum/apt 包管理器安装时，会自动在`/etc/logrotate.d/`目录下生成`nginx`配置文件；源码编译安装的 Nginx 需手动创建该配置文件。

#### 3.2.1 配置文件核心参数详解

表格

|参数|功能说明|生产环境推荐值|
|---|---|---|
|`daily/weekly/monthly`|切割周期，按日 / 周 / 月切割|高流量场景`daily`，低流量场景`weekly`|
|`rotate N`|保留归档日志的份数，超出份数的旧日志自动删除|30（保留 30 天），合规场景可调整至 90/180|
|`compress`|开启归档日志的 gzip 压缩，压缩比可达 10:1，大幅节省磁盘空间|开启`compress`|
|`delaycompress`|延迟压缩，切割后的第一个归档文件不压缩，下一次切割时再压缩|开启，避免切割时压缩大文件导致 IO 峰值|
|`missingok`|日志文件不存在时不报错，继续执行，避免定时任务异常|开启|
|`notifempty`|日志文件为空时不执行切割，避免生成无效空归档文件|开启|
|`create mode user group`|新建日志文件的权限、所有者、所属组|与 Nginx 运行用户匹配，如`create 640 nginx nginx`|
|`postrotate/endscript`|切割完成后执行的 Shell 脚本，核心是向 Nginx 发送 USR1 信号|必须配置，实现日志平滑切换|
|`sharedscripts`|多个日志文件匹配时，仅执行一次 postrotate 脚本，而非每个文件执行一次|开启，避免重复发送信号|
|`dateext`|归档文件添加日期后缀，替代默认的数字序号，便于日志溯源|开启|
|`dateformat`|日期后缀格式，如`-%Y%m%d`|推荐`-%Y%m%d`，按天切割|

#### 3.2.2 生产环境标准配置示例

##### 1. yum/apt 安装的 Nginx（默认路径）

conf

```
# 配置文件路径：/etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 640 nginx nginx
    sharedscripts
    dateext
    dateformat -%Y%m%d
    postrotate
        # 检查Nginx主进程是否存在，存在则发送USR1信号
        if [ -f /run/nginx.pid ]; then
            kill -USR1 `cat /run/nginx.pid`
        fi
    endscript
}
```

##### 2. 源码编译安装的 Nginx（自定义路径）

conf

```
# 配置文件路径：/etc/logrotate.d/nginx
/usr/local/nginx/logs/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 640 www www
    sharedscripts
    dateext
    dateformat -%Y%m%d
    postrotate
        # 源码安装的Nginx PID文件路径
        if [ -f /usr/local/nginx/logs/nginx.pid ]; then
            kill -USR1 `cat /usr/local/nginx/logs/nginx.pid`
        fi
    endscript
}
```

##### 3. 高流量场景小时级切割配置

对于日均日志量超过 10GB 的高流量场景，需配置小时级切割，避免单日志文件过大，影响查询与归档效率：

conf

```
# 配置文件路径：/etc/logrotate.d/nginx_hourly
/var/log/nginx/*.log {
    hourly
    rotate 720
    compress
    delaycompress
    missingok
    notifempty
    create 640 nginx nginx
    sharedscripts
    dateext
    dateformat -%Y%m%d%H
    postrotate
        if [ -f /run/nginx.pid ]; then
            kill -USR1 `cat /run/nginx.pid`
        fi
    endscript
}
```

> 小时级切割需额外配置定时任务：将 logrotate 执行脚本从`/etc/cron.daily/`复制到`/etc/cron.hourly/`，确保每小时执行一次。

### 3.3 配置验证与手动执行

1. **调试模式验证配置**：配置完成后，通过调试模式检查配置是否合法，无实际执行操作：
    
    bash
    
    运行
    
    ```
    logrotate -d /etc/logrotate.d/nginx
    ```
    
2. **强制手动触发切割**：可手动强制执行切割，验证流程是否正常，用于紧急场景或配置上线验证：
    
    bash
    
    运行
    
    ```
    logrotate -f /etc/logrotate.d/nginx
    ```
    
3. **执行结果验证**：执行后检查日志目录是否生成归档文件，Nginx 是否生成新的日志文件，新请求是否正常写入新日志，无报错信息。

### 3.4 常见陷阱与最佳实践

1. **PID 文件路径必须准确**：postrotate 脚本中的 PID 文件路径必须与 Nginx 配置文件中`pid`指令指定的路径完全一致，否则信号无法发送到 Nginx 主进程，导致切割后 Nginx 仍写入旧日志文件，是最常见的配置错误。
2. **权限约束**：logrotate 以 root 用户执行，归档目录必须对 root 用户有读写权限，新建日志文件的权限必须与 Nginx 运行用户匹配，否则会导致 Nginx 无法写入新日志。
3. **禁止使用 reload 替代 USR1**：`kill -HUP`是重载 Nginx 配置，会重新加载所有配置、重建 Worker 进程，属于重量级操作，可能导致业务短暂波动；`USR1`是专门用于日志重新打开的轻量级信号，是官方唯一推荐的日志切割信号。
4. **归档目录规划**：生产环境建议将日志归档到独立的磁盘分区，避免日志文件占满系统盘，导致操作系统异常；同时配置磁盘空间监控，当使用率超过 85% 时触发告警。
5. **合规性适配**：等保 2.0、金融行业合规标准要求日志至少保留 6 个月，需调整`rotate`参数至 180 以上，同时将归档日志备份到离线存储，满足审计要求。
6. **压缩性能优化**：高流量场景可配置`compresscmd /usr/bin/pigz`，使用多线程 gzip 压缩工具替代单线程 gzip，大幅降低压缩耗时与 CPU 开销。

---

## 04 基础监控看板 stub_status 模块

Nginx 的基础状态监控由**ngx_http_stub_status_module**标准模块提供，源码编译时需添加`--with-http_stub_status_module`参数启用，yum/apt 包管理器安装的 Nginx 默认已启用。该模块可暴露 Nginx 核心运行状态指标，包括连接数、请求总数、读写状态等，是服务健康检查、性能瓶颈定位、基础监控的核心手段，无需依赖任何第三方组件。

### 4.1 官方标准语法与配置

nginx

```
stub_status;
```

**合法作用域**：`server`、`location`

**核心功能**：在匹配的 location 中开启状态页面，输出 Nginx 核心运行指标。

#### 4.1.1 生产环境安全配置示例

状态页面包含服务核心运行数据，禁止公网无限制访问，必须配置 IP 白名单 + HTTP 基础认证双重防护：

nginx

```
server {
    listen 443 ssl http2;
    server_name status.example.com;
    ssl_certificate /etc/nginx/ssl/example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;

    # 状态监控专属location
    location /nginx_status {
        # 1. IP白名单：仅允许内网、运维办公网IP访问
        allow 127.0.0.1;
        allow 192.168.1.0/24;
        allow 223.xxx.xxx.xxx/32;
        deny all;

        # 2. HTTP基础认证，双重防护
        auth_basic "Nginx Status - Authorized Access Only";
        auth_basic_user_file /etc/nginx/auth_basic/status.htpasswd;

        # 3. 开启状态输出
        stub_status;

        # 4. 关闭日志记录，避免无效日志
        access_log off;
    }
}
```

配置完成后，重载 Nginx，通过`https://status.example.com/nginx_status`即可访问状态页面，输出格式如下：

plaintext

```
Active connections: 291
server accepts handled requests
 214852 214852 1054723
Reading: 6 Writing: 179 Waiting: 106
```

### 4.2 核心指标深度解析

状态页面的 7 个核心指标，覆盖 Nginx 连接层、请求层的全维度运行状态，是监控与性能分析的核心，必须精准理解每个指标的含义：

表格

|指标名称|指标含义|监控与分析价值|
|---|---|---|
|`Active connections`|当前活跃的客户端 TCP 连接总数，包括正在读写的连接与空闲等待的长连接|反映服务当前并发连接压力，数值持续超过服务器最大承载的 80% 时，需扩容或优化|
|`accepts`|Nginx 启动以来，累计接受的客户端 TCP 连接总数|累计连接量统计，用于容量规划、业务增长分析|
|`handled`|Nginx 启动以来，累计成功处理的 TCP 连接总数|正常情况下与`accepts`完全相等；若`handled`远小于`accepts`，说明服务器资源耗尽（文件描述符、内存不足），连接被丢弃，是严重故障信号|
|`requests`|Nginx 启动以来，累计处理的客户端 HTTP 请求总数|累计请求量统计，用于 QPS 计算、业务峰值分析；HTTP 长连接场景下，一个 TCP 连接可处理多个请求，因此`requests >= handled`|
|`Reading`|当前正在读取客户端请求头的 TCP 连接数|正常情况下数值很小；数值持续偏高，说明客户端上行带宽不足、请求头过大，或遭遇慢速连接攻击|
|`Writing`|当前正在向客户端发送响应的 TCP 连接数|数值持续偏高，说明响应体过大、客户端下行带宽不足，或服务器下行带宽达到瓶颈|
|`Waiting`|当前空闲的、等待客户端新请求的长连接数，也叫`keepalive`空闲连接|开启 HTTP 长连接后，该数值为正常空闲连接，数值不为 0 说明长连接复用正常；数值为 0 说明长连接未生效，每个请求都新建 TCP 连接，需优化`keepalive_timeout`参数|

### 4.3 基于指标的核心计算与分析

1. **QPS（每秒请求数）计算**：
    
    QPS 是服务核心吞吐指标，通过两次采样的`requests`差值除以采样间隔计算：
    
    plaintext
    
    ```
    QPS = (requests2 - requests1) / (time2 - time1)
    ```
    
    生产环境监控中，通常取 15 秒 / 1 分钟的采样间隔，计算实时 QPS。
    
2. **TCP 连接复用率计算**：
    
    反映 HTTP 长连接的复用效率，复用率越高，TCP 握手开销越小，服务性能越好：
    
    plaintext
    
    ```
    连接复用率 = requests / handled
    ```
    
    正常业务场景下，复用率应大于 5；静态资源站点应大于 10；若复用率接近 1，说明长连接未生效，需优化`keepalive`相关配置。
    
3. **请求处理成功率计算**：
    
    反映服务连接处理的稳定性：
    
    plaintext
    
    ```
    连接处理成功率 = handled / accepts * 100%
    ```
    
    正常情况下应等于 100%，低于 99.9% 时需排查服务器资源瓶颈、配置错误。
    

### 4.4 关键注意事项与最佳实践

1. **安全防护必须到位**：状态页面禁止公网无限制开放，必须配置 IP 白名单 + 基础认证，避免服务运行状态泄露，被攻击者用于定向攻击。
2. **指标生命周期**：所有累计指标（accepts、handled、requests）均为 Nginx 启动以来的累计值，重启后清零；监控系统必须计算速率指标（如 QPS、每秒新建连接数），而非直接监控累计值。
3. **健康检查适配**：该状态接口可作为 Nginx 服务的健康检查端点，负载均衡器可通过访问该接口判断 Nginx 服务是否存活，无需额外开发健康检查接口。
4. **权限约束**：状态页面的 HTTPS 证书、认证文件必须配置正确的权限，避免出现 403/401 错误，导致监控数据采集失败。
5. **采样频率规范**：基础监控采样间隔建议为 10-15 秒，避免采样过于频繁导致 Nginx 性能损耗，同时保证异常事件的及时捕获。

---

## 05 进阶监控 Prometheus + Nginx Exporter + Grafana 体系

企业级生产环境中，基础的 stub_status 模块无法满足长期时序存储、多维度可视化、异常告警、多实例集群监控的需求，**Prometheus + Nginx Exporter + Grafana**是行业标准的开源监控解决方案，可实现 Nginx 集群的全维度、可视化、可预警的全栈监控体系。

### 5.1 监控架构与核心组件角色

该体系分为三大核心组件，各司其职，形成完整的监控闭环：

表格

|组件名称|核心角色与功能|
|---|---|
|**Nginx Exporter**|部署在 Nginx 服务器上，负责抓取 Nginx stub_status 模块的指标，转换为 Prometheus 兼容的时序数据格式，暴露`/metrics`接口供 Prometheus 抓取|
|**Prometheus**|时序数据库，按照配置的抓取规则，定期从 Nginx Exporter 拉取监控指标，完成数据存储、时序计算，提供 PromQL 查询语言支持指标查询与聚合|
|**Grafana**|可视化与告警平台，对接 Prometheus 数据源，通过拖拽式面板实现监控指标的可视化展示，配置异常告警规则，支持邮件、钉钉、企业微信等多渠道告警通知|

### 5.2 部署与配置全流程

#### 5.2.1 前置条件

Nginx 服务器已启用`stub_status`模块，配置了可访问的状态接口（如`/nginx_status`），且 Nginx Exporter 服务器可访问该接口。

#### 5.2.2 Nginx Exporter 部署

Nginx Exporter 由 Prometheus 社区官方维护，推荐使用二进制部署或 Docker 部署两种方式。

##### 1. 二进制部署（生产环境推荐）

bash

运行

```
# 1. 下载最新版本（替换为最新release版本号）
wget https://github.com/nginxinc/nginx-prometheus-exporter/releases/download/v1.3.0/nginx-prometheus-exporter_1.3.0_linux_amd64.tar.gz
# 2. 解压安装
tar -zxvf nginx-prometheus-exporter_1.3.0_linux_amd64.tar.gz
mv nginx-prometheus-exporter /usr/local/bin/
# 3. 验证安装
nginx-prometheus-exporter --version
```

##### 2. 配置 Systemd 服务，实现开机自启

创建服务文件`/etc/systemd/system/nginx-exporter.service`：

ini

```
[Unit]
Description=Nginx Prometheus Exporter
After=network.target nginx.service

[Service]
Type=simple
User=nginx
# 核心启动参数：指定Nginx状态接口地址，监听端口默认9113
ExecStart=/usr/local/bin/nginx-prometheus-exporter \
    --nginx.scrape-uri=https://status.example.com/nginx_status \
    --web.listen-address=0.0.0.0:9113 \
    --nginx.ssl-verify=false
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

启动服务并设置开机自启：

bash

运行

```
systemctl daemon-reload
systemctl start nginx-exporter
systemctl enable nginx-exporter
# 验证服务状态
systemctl status nginx-exporter
# 验证metrics接口
curl http://127.0.0.1:9113/metrics
```

##### 3. Docker 部署

bash

运行

```
docker run -d \
  --name nginx-exporter \
  -p 9113:9113 \
  nginx/nginx-prometheus-exporter:latest \
  --nginx.scrape-uri=https://status.example.com/nginx_status
```

#### 5.2.3 Prometheus 配置

修改 Prometheus 配置文件`prometheus.yml`，添加 Nginx 监控抓取任务：

yaml

```
global:
  scrape_interval: 15s # 全局抓取间隔15秒
  evaluation_interval: 15s

scrape_configs:
  # Nginx集群监控任务
  - job_name: "nginx_cluster"
    static_configs:
      # 所有Nginx实例的Exporter地址
      - targets:
        - "192.168.1.10:9113"
        - "192.168.1.11:9113"
        - "192.168.1.12:9113"
        labels:
          env: "production"
          project: "web_service"
```

配置完成后，重启 Prometheus，在 Prometheus Web UI 的`Targets`页面中，可看到 Nginx 实例的状态为`UP`，说明抓取正常。

#### 5.2.4 Grafana 可视化配置

1. **添加 Prometheus 数据源**：登录 Grafana，进入「Configuration」→「Data Sources」→「Add data source」，选择 Prometheus，填写 Prometheus 服务地址，保存并测试连接。
2. **导入官方监控看板**：Grafana 官方市场提供了成熟的 Nginx 监控看板，推荐使用看板 ID：**12708**（Nginx Exporter 官方看板），进入「Dashboards」→「Import」，输入看板 ID，选择 Prometheus 数据源，点击导入即可。
3. **核心看板指标**：导入完成后，可看到完整的监控面板，核心指标包括：
    
    - 服务存活状态、实例在线数
    - 实时 QPS、请求量趋势
    - 活跃连接数、新建连接数趋势
    - 请求状态码分布、4xx/5xx 错误率
    - TCP 连接复用率、长连接空闲数
    - 请求读写状态分布、慢请求占比
    - 多实例集群维度的聚合统计
    

### 5.3 核心告警规则配置

生产环境必须配置核心告警规则，实现异常事件的及时通知，告警规则配置在 Prometheus 的`rules`文件中，核心告警规则示例如下：

yaml

```
groups:
- name: nginx_alerts
  rules:
  # 告警1：Nginx实例宕机，Exporter无法抓取
  - alert: NginxInstanceDown
    expr: up{job="nginx_cluster"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Nginx实例宕机"
      description: "实例 {{ $labels.instance }} 已超过1分钟无法抓取，服务可能宕机"

  # 告警2：5xx错误率过高，超过1%
  - alert: Nginx5xxErrorRateHigh
    expr: sum(rate(nginx_http_responses_total{code=~"5.."}[5m])) / sum(rate(nginx_http_responses_total[5m])) * 100 > 1
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "Nginx 5xx错误率过高"
      description: "实例 {{ $labels.instance }} 5xx错误率已达 {{ $value | printf \"%.2f\" }}%，超过阈值1%"

  # 告警3：QPS突降，超过50%
  - alert: NginxQpsDrop
    expr: (sum(rate(nginx_http_requests_total[5m])) / sum(rate(nginx_http_requests_total[5m] offset 10m)) - 1) * 100 < -50
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Nginx QPS突降"
      description: "集群QPS较10分钟前下降 {{ $value | printf \"%.2f\" }}%，业务可能异常"

  # 告警4：活跃连接数超过阈值
  - alert: NginxActiveConnectionsHigh
    expr: nginx_connections_active > 10000
    for: 3m
    labels:
      severity: warning
    annotations:
      summary: "Nginx活跃连接数过高"
      description: "实例 {{ $labels.instance }} 活跃连接数已达 {{ $value }}，超过阈值10000"
```

### 5.4 进阶监控扩展

1. **日志指标监控**：通过`filebeat`采集 Nginx 访问日志，提取请求耗时、状态码、接口维度的指标，输出到 Prometheus，实现接口级别的精细化监控，定位慢接口、异常接口。
2. **SSL 证书监控**：通过`blackbox_exporter`监控 HTTPS 证书有效期，提前 30 天触发证书过期告警，避免证书过期导致服务不可用。
3. **集群高可用监控**：针对 Nginx+Keepalived 高可用集群，监控 VIP 漂移状态、Keepalived 进程存活、主备节点状态，实现集群层面的高可用监控。
4. **多租户权限控制**：Grafana 中配置多租户权限，不同业务线的运维人员仅可查看对应 Nginx 实例的监控看板，满足企业级权限管控要求。

### 5.5 最佳实践与注意事项

1. **Exporter 安全防护**：Nginx Exporter 的 9113 端口必须通过防火墙限制访问，仅允许 Prometheus 服务器 IP 访问，禁止公网开放，避免监控指标泄露。
2. **抓取间隔规范**：生产环境抓取间隔建议设置为 15s，高频交易场景可调整为 10s，避免抓取过于频繁导致服务器性能损耗，同时保证异常事件的及时捕获。
3. **告警分级规范**：告警必须分级，`critical`级别（服务宕机、5xx 错误率飙升）仅发送给核心运维人员，`warning`级别仅工作时段通知，避免告警风暴。
4. **数据持久化**：Prometheus 建议配置远程存储（如 Thanos、M3DB），实现监控数据的长期存储，满足合规审计要求；同时配置 Prometheus 高可用集群，避免监控单点故障。
5. **全链路监控整合**：将 Nginx 监控与后端服务、数据库、服务器主机监控整合到同一 Grafana 看板，实现从接入层到应用层的全链路可观测，故障定位效率提升 10 倍以上。