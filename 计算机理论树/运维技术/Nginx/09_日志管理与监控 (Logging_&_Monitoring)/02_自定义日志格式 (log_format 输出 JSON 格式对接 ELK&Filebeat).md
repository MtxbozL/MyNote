
`log_format` 指令用于定义访问日志的输出格式，隶属于**ngx_http_log_module**标准模块，是实现日志结构化、字段自定义、对接第三方日志分析平台的核心指令。

### 2.1 官方标准语法与约束

```nginx
log_format name [escape=default|json|none] string ...;
```

**合法作用域**：仅可在`http`全局块中配置，禁止在`server`、`location`等子作用域中定义

**默认配置**：Nginx 内置`combined`通用日志格式，无需手动定义即可使用

> 核心约束：`log_format`必须在 http 全局块的顶层定义，所有 server 块、location 块仅可引用已定义的格式名称，无法在子作用域内新增或修改格式定义。

### 2.2 核心参数解析

1. **name 参数**：日志格式的自定义名称，全局唯一，区分大小写，用于`access_log`指令引用。
2. **escape 参数**：Nginx 1.11.8 版本新增，用于控制日志中特殊字符的转义规则，是结构化日志的核心配置：
    
| 取值        | 核心规则                                     | 适用场景                        |
| --------- | ---------------------------------------- | --------------------------- |
| `default` | 默认值，对特殊字符（双引号、反斜杠、换行、制表符）做 URL 编码转义      | 非结构化文本日志                    |
| `json`    | 按照 JSON 规范对特殊字符做转义，保证 JSON 格式的合法性，避免格式损坏 | 结构化 JSON 日志，对接 ELK/Filebeat |
| `none`    | 关闭所有转义，特殊字符原样输出                          | 自定义转义规则的特殊场景，生产环境不推荐        |
    
3. **string 参数**：日志格式的模板字符串，由固定文本与 Nginx 内置变量组成，多个字符串会自动拼接为完整的一行日志，每行日志对应一个客户端请求。

### 2.3 内置默认格式与核心变量

Nginx 内置`combined`通用日志格式，是绝大多数 Web 服务的标准格式，定义如下：

```nginx
log_format combined '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent"';
```

#### 2.3.1 核心基础变量解析

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

```nginx
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

```nginx
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
