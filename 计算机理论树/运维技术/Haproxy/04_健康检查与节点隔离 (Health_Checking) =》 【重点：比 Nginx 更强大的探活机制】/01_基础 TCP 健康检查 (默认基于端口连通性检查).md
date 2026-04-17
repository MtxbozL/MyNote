# 第四章 HAProxy 高可用体系与会话持久化机制

本章为 HAProxy 生产级落地的核心内容，完整定义其高可用架构的底层原理、健康检查体系、会话持久化机制、故障转移容灾能力与集群化部署方案，所有内容均基于 2.8.x LTS 版本官方标准，严格遵循高可用系统设计的学术规范，无冗余表述与非标准化配置，完整承接前三章的技术体系。

## 4.1 高可用核心设计理念与系统边界

### 4.1.1 核心定义

HAProxy 的高可用体系，是一套**以故障无感知为核心目标、多层级故障闭环为设计逻辑的端到端流量可靠性保障体系**。其核心解决两个本质问题：一是**单点故障规避**，通过故障自动隔离与转移，避免单个上游服务节点或 HAProxy 自身节点故障导致业务中断；二是**会话一致性保障**，确保有状态业务的客户端请求在生命周期内的转发一致性，避免业务逻辑异常。

### 4.1.2 核心设计原则

本体系严格遵循以下学术级高可用设计原则，所有特性均围绕该原则实现：

1. **故障快速感知原则**：通过主动 + 被动双维度检测，实现毫秒级故障节点识别，最小化故障窗口时长
2. **无状态转发优先原则**：核心转发逻辑无状态化，仅通过可同步的元数据实现会话保持，避免单点状态绑定
3. **故障自动闭环原则**：故障检测、隔离、转移、恢复全流程自动化，无需人工干预，最小化 MTTR（平均恢复时间）
4. **业务无感知原则**：所有高可用操作均在流量转发层面完成，对客户端与上游服务无侵入、无感知，不修改业务语义
5. **过载防护原则**：高可用操作必须附带限流与降级约束，避免故障转移导致的雪崩效应，保障系统整体稳定性
6. **可观测性原则**：所有高可用事件（故障隔离、节点恢复、主备切换、会话解绑）均有完整的日志与指标输出，可追溯、可监控

### 4.1.3 系统边界与能力分层

HAProxy 高可用体系分为四层能力边界，从底层到上层形成完整的可靠性保障链路，无交叉重叠：

表格

|能力层级|核心职责|核心实现|故障覆盖范围|
|---|---|---|---|
|L1 节点健康保障层|上游服务节点的故障检测与隔离|主动 / 被动健康检查体系|上游服务节点宕机、端口不可用、业务逻辑异常|
|L2 会话一致性层|客户端请求的转发亲和性保障|会话保持（持久化）机制|有状态业务的会话数据丢失、请求跨节点转发异常|
|L3 流量容灾转移层|故障场景下的流量自动调度与降级|请求重试、熔断、备份池、优雅上下线|上游集群部分 / 整体故障、流量过载、服务发布变更|
|L4 集群高可用层|HAProxy 自身单点故障规避|Keepalived+VRRP 主备集群、会话状态同步|HAProxy 单节点宕机、服务器硬件故障、机房网络故障|

## 4.2 上游节点健康检查体系

健康检查是高可用体系的底层基础，核心作用是**实时感知上游服务节点的可用性状态，自动隔离故障节点，恢复后自动重新接入流量**，彻底避免将请求转发到不可用的服务节点。HAProxy 提供全栈、可定制的健康检查能力，严格区分四层与七层检查体系，支持主动 + 被动双检测模式。

### 4.2.1 健康检查核心状态机

HAProxy 为每个上游服务节点维护独立的健康状态机，所有状态转换均有严格的触发条件，无模糊语义，完整状态流转如下：

表格

|状态名称|状态标识|核心定义|触发条件|流量处理规则||
|---|---|---|---|---|---|
|初始状态|NONE|节点启动后的初始状态，未执行任何健康检查|HAProxy 进程启动 / 重载，节点首次加载|不接收任何流量，等待首次健康检查完成||
|健康状态|UP|节点可用，可正常接收流量|连续`rise`次健康检查成功|按负载均衡算法与权重正常分发流量||
|可疑状态|UP|GOINGDOWN|节点出现检查失败，处于故障判定过渡期|健康检查失败，但未达到`fall`次阈值|继续接收流量，同步加快检查频率|
|故障状态|DOWN|节点不可用，被隔离，不接收任何流量|连续`fall`次健康检查失败，或被动检测触发故障判定|停止接收新请求，现有连接按规则处理，触发会话解绑||
|恢复状态|DOWN|GOINGUP|节点故障后，处于恢复验证过渡期|故障状态下，健康检查首次成功，但未达到`rise`次阈值|不接收业务流量，仅执行健康检查验证|

### 4.2.2 健康检查分类与核心架构

HAProxy 健康检查分为三大类，覆盖所有业务场景，各类检查的能力边界与性能开销严格区分：

表格

|检查类型|工作层级|核心原理|性能开销|故障感知精度|适用场景|
|---|---|---|---|---|---|
|主动四层 TCP 检查|传输层 L4|定期与上游节点建立 TCP 连接，通过三次握手成功与否判定可用性|极低|低，仅能验证端口连通性|TCP 服务负载均衡、底层网络连通性验证|
|主动七层应用检查|应用层 L7|定期向上游节点发送应用层请求（HTTP/MySQL/Redis 等），通过响应内容判定业务可用性|中|高，可验证业务逻辑可用性|HTTP 服务、有业务逻辑的应用层服务|
|被动业务流量检查|全层级|实时监控业务流量的转发结果，通过连接失败、请求超时、错误响应判定节点可用性|无额外开销|极高，基于真实业务流量判定|所有生产环境场景，作为主动检查的补充|

### 4.2.3 四层（TCP）健康检查完整规范

四层健康检查工作在 OSI 传输层，仅基于 TCP 协议交互完成可用性判定，无应用层解析开销，是 TCP 模式服务的默认检查方式。

#### 1. 基础 TCP 端口检查

- **核心语法**：在`server`指令中添加`check`参数启用基础 TCP 检查，配合核心参数完成配置
- **最小可用配置**：
    
    plaintext
    
    ```
    server mysql01 192.168.1.201:3306 check inter 2s rise 2 fall 3
    ```
    
- **核心执行逻辑**：HAProxy 每隔`inter`指定的间隔，与上游节点的指定端口建立 TCP 连接，三次握手成功则判定单次检查成功，连接超时 / 被拒绝则判定单次检查失败。

#### 2. 高级 TCP 内容检查（tcp-check 规则）

基础 TCP 检查仅能验证端口连通性，无法验证 TCP 服务的应用层可用性，`tcp-check`规则体系支持自定义 TCP 请求与响应匹配，实现对 MySQL、Redis、SSH 等 TCP 服务的业务级健康检查。

- **核心语法体系**：
    
    plaintext
    
    ```
    # 后端配置段中启用TCP内容检查
    option tcp-check
    # 定义检查规则，按书写顺序执行
    tcp-check send <发送内容>
    tcp-check expect <匹配规则> <匹配内容>
    ```
    
- **核心规则指令**：
    
    表格
    
    |指令|语法格式|核心作用|约束条件|
    |---|---|---|---|
    |`tcp-check send`|`tcp-check send <二进制/字符串内容>`|向上游节点发送指定的 TCP 数据包|支持字符串与十六进制二进制内容，字符串需用双引号包裹|
    |`tcp-check send-binary`|`tcp-check send-binary <十六进制内容>`|发送二进制格式的 TCP 数据包，适用于私有协议|十六进制内容无需前缀，直接书写，如`0d0a`|
    |`tcp-check expect`|`tcp-check expect <匹配模式> <匹配内容>`|匹配上游节点返回的 TCP 响应内容，匹配失败则判定检查失败|支持`string`/`rstring`/`binary`匹配模式，可搭配`-i`忽略大小写|
    |`tcp-check connect`|`tcp-check connect [port <端口>] [addr <地址>]`|自定义健康检查的连接地址与端口，默认使用 server 指令的地址端口|可实现对非业务端口的管理端口健康检查|
    |`tcp-check close`|`tcp-check close`|主动关闭 TCP 连接，结束本次健康检查|可选指令，默认检查完成后自动关闭连接|
    
- **生产级示例 1：Redis 服务健康检查**
    
    plaintext
    
    ```
    backend redis_back
        mode tcp
        balance leastconn
        option tcp-check
        # 发送PING命令
        tcp-check send "PING\r\n"
        # 期望返回+PONG响应
        tcp-check expect string "+PONG"
        # 服务器节点配置
        server redis01 192.168.1.210:6379 check inter 1s rise 2 fall 3
        server redis02 192.168.1.211:6379 check inter 1s rise 2 fall 3 backup
    ```
    
- **生产级示例 2：MySQL 服务健康检查**
    
    plaintext
    
    ```
    backend mysql_back
        mode tcp
        balance leastconn
        option tcp-check
        # MySQL初始握手包匹配，检查返回的MySQL协议版本
        tcp-check expect rstring "5\\.[5-7]\\.[0-9]+"
        # 服务器节点配置
        server mysql01 192.168.1.201:3306 check inter 2s rise 2 fall 3 port 3306
    ```
    

### 4.2.4 七层（HTTP/HTTPS）健康检查完整规范

七层健康检查工作在 OSI 应用层，通过发送 HTTP 请求并验证响应内容，判定上游 Web 服务的业务可用性，是 HTTP 模式服务的核心检查方式，可精准识别「端口通但业务不可用」的假活状态。

#### 1. 基础 HTTP 健康检查

- **核心语法**：通过`option httpchk`启用基础 HTTP 检查，配合`server`指令的`check`参数生效
    
- **标准语法格式**：
    
    plaintext
    
    ```
    option httpchk <HTTP方法> <请求URI> <HTTP版本>
    ```
    
    - `HTTP方法`：推荐使用`OPTIONS`/`HEAD`，无请求体开销，性能最优；需业务验证时使用`GET`
    - `请求URI`：健康检查的请求路径，推荐使用专用的健康检查接口，如`/health/check`
    - `HTTP版本`：推荐`HTTP/1.1`，需配合`Host`请求头，兼容虚拟主机场景
    
- **核心执行逻辑**：HAProxy 定期与上游节点建立 TCP 连接，发送预设的 HTTP 请求，通过响应状态码判定检查结果：默认 2xx/3xx 状态码为检查成功，4xx/5xx 为检查失败。
    
- **生产级基础配置示例**：
    
    plaintext
    
    ```
    backend web_back
        mode http
        balance roundrobin
        # 启用HTTP健康检查，发送GET请求到/health接口，使用HTTP/1.1协议
        option httpchk GET /health HTTP/1.1
        # 添加Host请求头，兼容虚拟主机
        http-check send hdr Host www.example.com
        # 服务器节点配置
        server web01 192.168.1.101:80 check inter 2s rise 2 fall 3
        server web02 192.168.1.102:80 check inter 2s rise 2 fall 3
    ```
    

#### 2. 高级 HTTP 内容检查（http-check 规则）

2.0 + 版本提供的`http-check`规则体系，支持自定义 HTTP 请求头、请求体、响应匹配规则，可实现基于响应内容、响应头、状态码的精细化健康检查，是生产环境推荐用法。

- **核心语法体系**：
    
    plaintext
    
    ```
    # 启用HTTP健康检查
    option httpchk
    # 定义请求规则
    http-check send [meth <方法>] [uri <URI>] [ver <版本>] [hdr <名称> <值>] [body <内容>]
    # 定义响应匹配规则
    http-check expect <匹配模式> <匹配条件>
    ```
    
- **核心匹配模式**：
    
    表格
    
    |匹配模式|核心作用|匹配成功条件|
    |---|---|---|
    |`status`|匹配 HTTP 响应状态码|响应状态码符合预设条件，支持区间匹配，如`status 200-399`|
    |`rstatus`|正则匹配响应状态码|响应状态码匹配预设正则表达式|
    |`string`|精确匹配响应体内容|响应体包含预设的完整字符串|
    |`rstring`|正则匹配响应体内容|响应体匹配预设的正则表达式|
    |`hdr`|匹配响应头内容|指定响应头的值符合预设匹配条件|
    |`rhdr`|正则匹配响应头内容|指定响应头的值匹配预设正则表达式|
    
- **生产级高级配置示例**：
    
    plaintext
    
    ```
    backend api_back
        mode http
        balance roundrobin
        option httpchk
        # 定义健康检查请求：POST方法，JSON请求体，携带认证头
        http-check send meth POST uri /api/health ver HTTP/1.1 \
            hdr Host api.example.com \
            hdr Content-Type application/json \
            body '{"check":"system"}'
        # 定义响应匹配规则：状态码必须为200，响应体必须包含"status":"ok"
        http-check expect status 200
        http-check expect string '"status":"ok"'
        # 服务器节点配置
        server api01 192.168.1.151:8080 check inter 2s rise 2 fall 3
        server api02 192.168.1.152:8080 check inter 2s rise 2 fall 3
    ```
    

#### 3. HTTPS 健康检查

针对上游 HTTPS 服务，需在`server`指令中添加`ssl`参数，启用 TLS 加密的健康检查，兼容所有 HTTP 检查规则：

plaintext

```
backend https_back
    mode http
    option httpchk GET /health HTTP/1.1
    http-check send hdr Host www.example.com
    http-check expect status 200
    # 启用HTTPS健康检查，ssl参数开启TLS，verify none跳过证书校验（内网场景）
    server web01 192.168.1.101:443 check ssl verify none inter 2s rise 2 fall 3
```

### 4.2.5 被动健康检查机制

主动健康检查存在固定的检查间隔，无法实时感知业务流量中的突发故障，被动健康检查（也叫流量观测检查）通过监控真实业务流量的转发结果，实时判定节点可用性，无额外性能开销，是主动检查的必要补充。

- **核心语法**：`observe <模式>`，配合`on-error <动作>`、`error-limit <阈值>`参数使用
    
- **核心模式**：
    
    表格
    
    |模式|适用场景|核心观测范围|
    |---|---|---|
    |`layer4`|TCP 模式|观测 TCP 连接失败、连接超时、重置等四层错误|
    |`layer7`|HTTP 模式|观测 HTTP 请求超时、5xx 响应、响应异常等七层错误|
    
- **核心错误处理动作**：
    
    表格
    
    |动作|核心作用|适用场景|
    |---|---|---|
    |`fastinter`|触发后立即缩短健康检查间隔，快速验证节点状态|通用场景，平衡故障感知速度与检查开销|
    |`fail-check`|触发后直接计为一次健康检查失败，加速故障判定|高可靠性要求场景，快速隔离故障节点|
    |`mark-down`|触发后直接将节点标记为 DOWN，立即隔离|核心业务场景，零容忍业务错误|
    |`sudden-death`|触发后直接进入故障判定流程，单次错误即可标记为 DOWN|金融级高可用场景，极端严格的故障隔离|
    
- **生产级配置示例**：
    
    plaintext
    
    ```
    backend web_back
        mode http
        balance roundrobin
        option httpchk
        http-check send meth GET uri /health ver HTTP/1.1 hdr Host www.example.com
        http-check expect status 200
        # 启用七层被动健康检查，观测业务流量的HTTP错误
        observe layer7
        # 单次5xx错误计为一次健康检查失败，加速故障隔离
        on-error fail-check
        # 连续10次业务错误直接标记节点为DOWN
        error-limit 10
        on-marked-down shutdown-sessions
        # 服务器节点配置
        server web01 192.168.1.101:80 check inter 2s rise 2 fall 3
    ```
    

### 4.2.6 自定义外部脚本健康检查

对于需要复杂业务逻辑验证的场景（如数据库主从状态、集群节点角色、业务依赖服务可用性），HAProxy 支持通过外部脚本执行自定义健康检查，通过脚本返回值判定节点可用性。

- **核心启用条件**：`global`段中必须配置`external-check`指令，开启外部脚本检查能力
    
- **核心语法**：`external-check command <脚本路径>`，配合`external-check path`指定脚本搜索路径
    
- **执行规则**：
    
    1. 每次健康检查时，HAProxy 会执行指定脚本，传入以下参数：`脚本路径 <服务器IP> <服务器端口> <服务器名称> <后端名称>`
    2. 脚本返回值为 0，判定检查成功；返回值非 0，判定检查失败
    3. 脚本执行超时时间与`timeout check`绑定，超时则判定检查失败
    
- **安全约束**：脚本必须为 HAProxy 运行用户可执行，禁止赋予 root 权限，避免命令注入风险，生产环境需严格管控脚本内容与权限。
    
- **生产级配置示例**：
    
    plaintext
    
    ```
    # global段开启外部检查能力
    global
        external-check
        external-check path /etc/haproxy/scripts/
    
    # 后端配置段
    backend mysql_master_back
        mode tcp
        balance leastconn
        option external-check
        # 指定健康检查脚本
        external-check command mysql_master_check.sh
        # 服务器节点配置
        server mysql01 192.168.1.201:3306 check inter 5s rise 2 fall 3
    ```
    

### 4.2.7 健康检查核心参数全解析

所有健康检查相关参数均有严格的语义定义，无模糊配置，生产环境核心参数如下：

表格

|参数|语法格式|核心作用|推荐取值|约束条件|
|---|---|---|---|---|
|`check`|`server`指令中添加`check`|为该节点启用主动健康检查|必选参数|未添加该参数的节点，仅在 TCP 连接失败时被动标记为 DOWN|
|`inter`|`inter <时间>`|健康状态下的检查间隔|2s-5s|故障状态下默认使用该值，可通过`downinter`单独配置|
|`fall`|`fall <次数>`|标记健康节点为 DOWN 所需的连续检查失败次数|3 次|取值≥1，次数越少故障感知越快，误判风险越高|
|`rise`|`rise <次数>`|标记故障节点为 UP 所需的连续检查成功次数|2-3 次|取值≥1，次数越少恢复越快，抖动风险越高|
|`timeout check`|`timeout check <时间>`|单次健康检查的超时时间|1s-2s|必须小于`inter`间隔，超时则判定本次检查失败|
|`port`|`port <端口号>`|健康检查使用的端口，可与业务端口分离|业务管理端口|适用于业务端口不对外暴露，仅管理端口提供健康检查的场景|
|`addr`|`addr <IP地址>`|健康检查使用的 IP 地址，可与业务 IP 分离|管理 IP 地址|适用于跨网络隔离的健康检查场景|
|`downinter`|`downinter <时间>`|故障状态下的检查间隔|1s|小于`inter`值，加快故障节点恢复验证|
|`fastinter`|`fastinter <时间>`|可疑状态下的检查间隔|500ms|远小于`inter`值，加速故障状态确认|
|`slowstart`|`slowstart <时间>`|节点恢复后的慢启动时长，权重线性增长|30s-2m|避免节点恢复后瞬间接收全量流量导致过载，生产环境必配|
|`on-marked-down`|`on-marked-down <动作>`|节点被标记为 DOWN 时触发的动作|`shutdown-sessions`|`shutdown-sessions`会立即关闭该节点的所有现有连接，强制触发会话重分发|
|`on-marked-up`|`on-marked-up <动作>`|节点被标记为 UP 时触发的动作|可选日志告警动作|用于节点恢复的事件通知|

### 4.2.8 健康检查性能优化与最佳实践

1. **检查间隔平衡原则**：核心业务使用短间隔（1-2s）加快故障感知，非核心业务使用长间隔（5-10s）减少上游服务压力
2. **检查请求轻量化原则**：优先使用 OPTIONS/HEAD 方法，避免使用带复杂业务逻辑的 GET/POST 请求，最小化健康检查对上游服务的性能影响
3. **主动 + 被动双检查原则**：所有生产环境节点必须同时配置主动检查与被动检查，兼顾故障感知速度与无额外开销
4. **专用检查接口原则**：HTTP 服务必须使用专用的健康检查接口，而非首页或业务接口，避免缓存、鉴权等因素导致的误判
5. **检查隔离原则**：健康检查使用独立的端口 / 接口，与业务流量隔离，避免业务流量过载导致的健康检查误判
6. **日志全量记录原则**：开启健康检查日志，所有节点上下线事件必须完整记录，用于故障追溯与审计

## 4.3 会话保持（持久化）机制

会话保持（也叫会话亲和性、会话持久化），是指**HAProxy 将同一客户端的多个请求，始终转发到同一台上游服务节点的机制**，核心解决有状态业务的会话数据共享问题，避免因请求跨节点转发导致的登录状态丢失、会话数据失效、业务逻辑异常等问题。

### 4.3.1 会话保持核心设计原则

1. **最小侵入原则**：优先使用无侵入式会话保持方案，对客户端与上游服务无修改要求
2. **故障自动解绑原则**：会话绑定的节点故障后，必须自动解绑会话，将请求转发到健康节点，避免业务中断
3. **可配置粒度原则**：支持多维度的会话绑定粒度，从 IP 级到用户级，适配不同业务场景
4. **无状态优先原则**：优先使用无状态哈希算法实现会话保持，避免本地状态存储导致的集群会话不一致
5. **超时可控原则**：所有会话保持规则必须配置明确的超时时间，避免永久会话绑定导致的负载不均

### 4.3.2 四层会话保持机制

四层会话保持工作在 OSI 传输层，基于 TCP 连接的元数据实现会话绑定，无应用层解析开销，适用于 TCP 模式服务与无 Cookie 的 HTTP 场景。

#### 1. 源 IP 哈希会话保持

- **核心原理**：基于客户端源 IP 地址进行哈希计算，将同一源 IP 的所有请求始终转发到同一台上游节点，哈希环随后端节点列表动态调整
- **核心语法**：`balance source`，配合哈希参数调整算法行为
- **核心配置示例**：
    
    plaintext
    
    ```
    backend mysql_back
        mode tcp
        # 启用源IP哈希会话保持
        balance source
        # 哈希算法参数：一致性哈希，节点变化时最小化哈希重分布
        hash-type consistent
        server mysql01 192.168.1.201:3306 check
        server mysql02 192.168.1.202:3306 check
    ```
    
- **核心参数**：
    
    表格
    
    |参数|核心作用|适用场景|
    |---|---|---|
    |`hash-type consistent`|启用一致性哈希算法，节点上下线时仅影响 1/N 的哈希映射，最小化会话失效|容器化动态扩缩容场景，节点频繁变化的环境|
    |`hash-type map-based`|启用静态哈希表算法，哈希映射均匀度更高，但节点变化时全量重分布|节点固定不变的静态集群场景|
    
- **优缺点分析**：
    
    - 优势：无侵入性，兼容所有 TCP 协议，无需客户端与服务端做任何修改，配置简单
    - 劣势：NAT 环境下大量客户端共享同一源 IP，会导致负载不均；客户端 IP 变化会导致会话失效；节点变化会导致哈希环重分布，大量会话失效
    

#### 2. Stick-Table 四层会话保持

基于 Sticky Table 粘性表实现的四层会话保持，可自定义会话绑定维度与超时时间，支持故障自动解绑，是比源 IP 哈希更灵活的四层会话保持方案。

- **核心配置示例**：
    
    plaintext
    
    ```
    backend tcp_back
        mode tcp
        balance leastconn
        # 定义粘性表：基于IP存储，最大10万条记录，30分钟过期
        stick-table type ip size 100k expire 30m
        # 基于源IP进行会话绑定
        stick on src
        server node01 192.168.1.100:8080 check
        server node02 192.168.1.101:8080 check
    ```
    
- **核心优势**：支持自定义会话超时时间，可通过`peers`段实现集群间会话同步，节点故障时自动解绑会话，负载均衡算法与会话保持解耦，可同时使用最小连接数算法保证负载均衡。

### 4.3.3 七层 HTTP 会话保持机制

七层会话保持工作在 OSI 应用层，基于 HTTP 请求的特征实现会话绑定，亲和性精度更高，适配绝大多数 Web 业务场景，是 HAProxy 的核心竞争力之一。分为两大类：**Cookie-based 会话保持**与**Stick-Table 特征会话保持**。

#### 1. Cookie-based 会话保持

基于 HTTP Cookie 实现会话绑定，是 Web 业务最常用的会话保持方案，精度可到用户级，无负载不均问题，支持故障自动解绑。HAProxy 提供四种 Cookie 会话保持模式，覆盖所有业务场景。

##### 模式 1：植入式 Cookie（insert 模式）

- **核心原理**：HAProxy 在首次响应客户端时，植入一个自定义 Cookie，记录目标服务器的标识；后续客户端请求携带该 Cookie 时，HAProxy 直接将请求转发到对应的服务器节点
- **核心优势**：对上游服务无侵入，无需服务端修改任何代码，Cookie 由 HAProxy 全权管理，支持加密防篡改
- **生产级配置示例**：
    
    plaintext
    
    ```
    backend web_back
        mode http
        balance roundrobin
        # 启用植入式Cookie会话保持，Cookie名称为SERVERID，nocache避免缓存，indirect隐藏Cookie值
        cookie SERVERID insert nocache indirect maxidle 3600s maxlife 86400s
        # 服务器节点配置，cookie参数指定节点对应的Cookie值
        server web01 192.168.1.101:80 check cookie web01
        server web02 192.168.1.102:80 check cookie web02
    ```
    
- **核心参数详解**：
    
    表格
    
    |参数|核心作用|生产环境建议|
    |---|---|---|
    |`insert`|启用植入模式，HAProxy 自动向响应中注入 Cookie|必选参数|
    |`nocache`|禁止缓存带该 Cookie 的响应，避免代理服务器缓存导致的 Cookie 分发异常|必选配置|
    |`indirect`|隐藏 Cookie 值，向上游服务隐藏 HAProxy 注入的会话 Cookie，避免业务逻辑干扰|推荐配置|
    |`preserve`|客户端已携带有效 Cookie 时，不修改 Cookie，保留原有值|推荐配置|
    |`maxidle <时间>`|Cookie 的最大空闲时长，超过该时间无请求则 Cookie 失效|推荐 3600s|
    |`maxlife <时间>`|Cookie 的最大生命周期，无论是否有请求，到期自动失效|推荐 86400s|
    |`secure`|标记 Cookie 为 Secure，仅 HTTPS 请求携带|HTTPS 场景必选配置|
    |`httponly`|标记 Cookie 为 HttpOnly，禁止前端 JS 读取，防止 XSS 攻击|推荐配置|
    |`domain <域名>`|指定 Cookie 的生效域名|多域名场景必选配置|
    

##### 模式 2：前缀式 Cookie（prefix 模式）

- **核心原理**：HAProxy 不注入新 Cookie，而是在服务端原有 Cookie 前添加服务器标识前缀，后续请求中自动剥离前缀，转发到对应节点
- **核心优势**：兼容禁用第三方 Cookie 的场景，对服务端仅需少量适配，无额外 Cookie 开销
- **配置示例**：
    
    plaintext
    
    ```
    backend web_back
        mode http
        balance roundrobin
        # 启用前缀模式，基于服务端的SESSIONID Cookie添加前缀
        cookie SESSIONID prefix nocache
        server web01 192.168.1.101:80 check cookie web01
        server web02 192.168.1.102:80 check cookie web02
    ```
    

##### 模式 3：重写式 Cookie（rewrite 模式）

- **核心原理**：HAProxy 重写服务端原有 Cookie 的值，嵌入服务器标识，后续请求中还原 Cookie 值并转发到对应节点
- **适用场景**：服务端对 Cookie 名称有严格要求，不允许新增 Cookie 或修改前缀的场景
- **劣势**：对服务端有侵入，需适配 Cookie 值的重写规则，生产环境使用较少

##### 模式 4：被动式 Cookie（existing 模式）

- **核心原理**：HAProxy 不修改任何 Cookie，仅匹配客户端请求中已有的 Cookie 值，将指定 Cookie 值的请求转发到对应节点
- **核心优势**：完全无侵入，对客户端与服务端无任何修改，完全基于业务现有 Cookie 实现会话保持
- **适用场景**：服务端已生成唯一用户会话 Cookie，需基于该 Cookie 实现会话绑定的场景
- **配置示例**：
    
    plaintext
    
    ```
    backend web_back
        mode http
        balance roundrobin
        # 启用被动模式，基于服务端的USER_SESSION Cookie匹配
        cookie USER_SESSION existing nocache
        server web01 192.168.1.101:80 check cookie user01
        server web02 192.168.1.102:80 check cookie user02
    ```
    

#### 2. Stick-Table 七层特征会话保持

基于 Sticky Table 粘性表实现的七层会话保持，可基于 HTTP 请求的任意特征（Cookie、请求头、URL、用户 ID 等）实现会话绑定，灵活性极高，支持集群间会话同步，是高级业务场景的首选方案。

- **核心原理**：HAProxy 从 HTTP 请求中提取指定特征值，存入粘性表中，记录该特征对应的上游服务器节点；后续请求提取到相同特征值时，直接转发到记录的节点，实现会话保持
    
- **核心语法体系**：
    
    1. 定义粘性表：`stick-table type <特征类型> size <容量> expire <超时时间> [store <存储字段>]`
    2. 定义绑定规则：`stick on <提取因子> <table <粘性表名称>>`
    
- **生产级配置示例 1：基于 Cookie 的会话保持**
    
    plaintext
    
    ```
    backend web_back
        mode http
        balance roundrobin
        # 定义粘性表：基于字符串类型（Cookie值），最大10万条，30分钟过期
        stick-table type string size 100k expire 30m
        # 基于SESSIONID Cookie的值进行会话绑定
        stick on cookie(SESSIONID)
        server web01 192.168.1.101:80 check
        server web02 192.168.1.102:80 check
    ```
    
- **生产级配置示例 2：基于请求头的会话保持**
    
    plaintext
    
    ```
    backend api_back
        mode http
        balance roundrobin
        # 定义粘性表：基于字符串类型（用户ID），最大100万条，1小时过期
        stick-table type string size 1m expire 1h
        # 基于X-User-ID请求头的值进行会话绑定
        stick on hdr(X-User-ID)
        server api01 192.168.1.151:8080 check
        server api02 192.168.1.152:8080 check
    ```
    
- **核心优势**：
    
    - 完全自定义绑定维度，支持任意 HTTP 特征提取，适配所有业务场景
    - 会话保持与负载均衡算法解耦，可同时使用最优的负载均衡算法保证负载均匀
    - 支持集群间通过`peers`段实现粘性表数据同步，主备切换后会话不失效
    - 支持精细化的超时控制，可基于请求频率、连接数动态调整会话生命周期
    

### 4.3.4 会话保持的故障解绑机制

会话保持的核心风险是「节点故障后，会话仍绑定到故障节点，导致业务请求失败」，HAProxy 提供完整的故障自动解绑机制，彻底解决该问题。

#### 1. 核心解绑规则

1. **强制重分发机制**：`option redispatch`，当会话绑定的节点故障时，HAProxy 自动忽略会话保持规则，将请求重新分发到健康节点，是生产环境必配参数
    
    plaintext
    
    ```
    defaults
        mode http
        option redispatch
        retries 3
    ```
    
2. **会话强制失效机制**：`on-marked-down shutdown-sessions`，节点被标记为 DOWN 时，立即关闭该节点的所有现有连接，强制触发会话重分发，避免长连接会话持续绑定到故障节点
3. **粘性表自动清理机制**：节点被标记为 DOWN 后，粘性表中该节点对应的所有会话记录自动失效，后续请求重新选择健康节点并建立新的会话绑定
4. **Cookie 会话自动降级机制**：Cookie 模式下，当绑定的节点故障时，HAProxy 自动忽略客户端携带的 Cookie，重新选择健康节点，并向客户端注入新的会话 Cookie，实现无感知会话切换

#### 2. 会话保持的优雅降级规则

- `option persist`：当会话绑定的节点处于维护状态时，允许现有会话继续转发到该节点，新会话不绑定，适用于服务优雅下线场景
- `option force-persist`：强制保持会话绑定，即使节点处于故障状态，仅用于极端调试场景，生产环境禁止使用

### 4.3.5 各类会话保持方式技术对比

表格

|会话保持方式|工作层级|亲和性粒度|客户端兼容性|服务端侵入性|故障解绑能力|集群同步能力|性能开销|最佳适用场景|
|---|---|---|---|---|---|---|---|---|
|源 IP 哈希|L4|IP 级|全兼容|无|弱，节点变化会话全量失效|不支持|极低|TCP 服务、无 Cookie 的 HTTP 场景、NAT 环境少的内网场景|
|Stick-Table 四层|L4|自定义元数据级|全兼容|无|强，故障自动解绑|支持|低|TCP 服务、需自定义会话维度、集群高可用场景|
|Cookie 植入式|L7|用户级|高，兼容所有支持 Cookie 的浏览器|无|极强，自动重分发 + Cookie 更新|无需同步，无状态|中|通用 Web 业务、电商、后台管理系统，生产环境首选|
|Cookie 被动式|L7|用户级|极高，基于业务现有 Cookie|无|强，自动重分发|无需同步|低|已有成熟用户会话体系的业务、无侵入需求的场景|
|Stick-Table 七层|L7|自定义特征级（用户 / 接口 / 设备等）|全兼容|无|极强，故障自动解绑|支持|中|微服务、API 网关、需精细化会话绑定的高级业务场景|

## 4.4 故障转移与容灾降级机制

故障转移与容灾降级是高可用体系的闭环，核心解决**上游集群部分 / 整体故障时，如何保证业务的连续性，避免雪崩效应**，HAProxy 提供全链路的容灾能力，从请求级重试到集群级容灾切换，覆盖所有故障场景。

### 4.4.1 请求重试机制

请求重试是故障转移的第一道防线，核心作用是**当请求转发失败时，自动将请求重新分发到其他健康节点，避免单次请求失败，对客户端无感知**。

#### 1. 核心语法与配置

- **基础重试次数**：`retries <次数>`，定义单次请求的最大重试次数，推荐 2-3 次，避免过度重试导致雪崩
- **重试触发条件**：`retry-on <触发类型>`，定义哪些场景下触发重试，2.0 + 版本支持精细化的重试条件配置
- **生产级配置示例**：
    
    plaintext
    
    ```
    defaults
        mode http
        # 最大重试次数
        retries 3
        # 定义重试触发条件
        retry-on conn-failure timeout 500 5xx empty-response
        # 启用重分发，重试时忽略会话保持规则
        option redispatch
    ```
    

#### 2. 核心重试触发类型

表格

|触发类型|触发场景|幂等性风险|生产环境建议|
|---|---|---|---|
|`conn-failure`|与上游节点 TCP 连接失败|无风险，请求未发送到服务端|必选配置|
|`timeout`|连接超时 / 请求响应超时，可搭配`<最小超时时间>`，仅当超时超过该值时重试|低风险，需确认业务幂等性|推荐配置，建议设置最小超时时间 500ms|
|`5xx`|上游节点返回 5xx 状态码|高风险，请求已到达服务端，可能已执行业务逻辑|仅幂等接口启用，非幂等接口禁止使用|
|`404`|上游节点返回 404 状态码|低风险|仅容灾场景启用|
|`empty-response`|上游节点返回空响应|无风险，响应无效|推荐配置|
|`invalid-response`|上游节点返回无效的 HTTP 响应|无风险，响应无法解析|推荐配置|

#### 3. 重试安全约束与最佳实践

1. **幂等性原则**：仅对幂等接口（GET/HEAD/OPTIONS）启用重试，非幂等接口（POST/PUT/DELETE）禁止启用业务逻辑相关的重试，仅可启用连接失败类重试，避免重复提交
2. **重试次数限制原则**：最大重试次数不超过 3 次，避免故障节点压力持续叠加，导致雪崩
3. **超时阈值原则**：超时重试必须设置最小超时阈值，避免短超时频繁重试，加剧服务端压力
4. **重试隔离原则**：重试时必须启用`option redispatch`，强制将重试请求分发到其他健康节点，禁止重试到同一故障节点
5. **熔断保护原则**：重试必须配合熔断机制，当节点错误率超过阈值时，停止向该节点转发请求，避免持续重试

### 4.4.2 服务熔断与过载防护机制

熔断机制的核心作用是**当上游节点 / 集群的错误率、响应时长超过阈值时，自动隔离故障节点 / 集群，停止向其转发流量，避免故障扩散导致的系统雪崩**，是高可用体系的核心过载防护能力。

#### 1. 节点级熔断机制

基于健康检查与被动观测实现的节点级熔断，核心配置如下：

plaintext

```
backend web_back
    mode http
    balance roundrobin
    option httpchk GET /health HTTP/1.1
    http-check expect status 200
    # 被动健康检查，观测七层业务错误
    observe layer7
    # 单次5xx错误计为一次健康检查失败
    on-error fail-check
    # 连续5次错误直接标记节点为DOWN，触发熔断
    error-limit 5
    on-marked-down shutdown-sessions
    # 节点恢复后慢启动，60秒内权重线性增长，避免流量冲击
    server web01 192.168.1.101:80 check inter 2s rise 3 fall 3 slowstart 60s
    server web02 192.168.1.102:80 check inter 2s rise 3 fall 3 slowstart 60s
```

#### 2. 集群级容灾备份池机制

当主集群的健康节点数量低于阈值时，自动将流量切换到备份集群，实现集群级容灾，核心配置分为两种模式：

##### 模式 1：节点级备份

- **核心原理**：标记部分节点为`backup`备份节点，仅当所有非备份节点故障时，才会向备份节点转发流量
- **配置示例**：
    
    plaintext
    
    ```
    backend web_back
        mode http
        balance roundrobin
        # 主节点
        server web01 192.168.1.101:80 check
        server web02 192.168.1.102:80 check
        # 备份节点，主节点全部故障时启用
        server backup01 192.168.2.101:80 check backup
        server backup02 192.168.2.102:80 check backup
    ```
    

##### 模式 2：集群级备份切换

- **核心原理**：基于 ACL 匹配主集群的健康节点数量，当低于阈值时，通过`use_backend`自动切换到备份集群，灵活性更高，支持自定义切换阈值
- **生产级配置示例**：
    
    plaintext
    
    ```
    frontend http_front
        bind 0.0.0.0:80
        mode http
        default_backend main_web_back
        # 定义容灾ACL：主集群健康节点数小于2
        acl main_cluster_down nbsrv(main_web_back) lt 2
        # 备份集群可用
        acl backup_cluster_available nbsrv(backup_web_back) ge 1
        # 主集群故障时切换到备份集群
        use_backend backup_web_back if main_cluster_down backup_cluster_available
    
    # 主集群
    backend main_web_back
        mode http
        balance roundrobin
        option httpchk GET /health HTTP/1.1
        http-check expect status 200
        server web01 192.168.1.101:80 check
        server web02 192.168.1.102:80 check
    
    # 备份集群（跨机房容灾）
    backend backup_web_back
        mode http
        balance roundrobin
        option httpchk GET /health HTTP/1.1
        http-check expect status 200
        server backup01 192.168.2.101:80 check
        server backup02 192.168.2.102:80 check
    ```
    

#### 3. 服务降级机制

当系统过载时，自动降级非核心业务，保障核心业务的可用性，HAProxy 可通过 ACL 与流量管控实现精细化降级：

- **静态资源降级**：过载时将静态资源请求转发到 CDN 或静态缓存服务器
- **非核心接口降级**：过载时对非核心接口返回缓存数据或降级响应
- **流量削峰降级**：过载时对非核心用户限流，保障核心用户的服务可用性

### 4.4.3 优雅上下线与无损变更机制

生产环境中，服务发布、节点维护是高频操作，HAProxy 提供完整的优雅上下线机制，实现服务变更的零中断、无感知。

#### 1. 优雅下线机制

- **drain 排水模式**：将节点设置为 drain 模式，该节点仅接收已绑定的会话保持请求，不接收新请求，待现有连接处理完成后，即可下线维护
    
    - 运行时设置命令（通过 stats socket）：`set server web_back/web01 state drain`
    
- **maint 维护模式**：将节点设置为 maint 模式，立即停止接收所有请求，关闭现有连接，适用于紧急下线场景
    
    - 运行时设置命令：`set server web_back/web01 state maint`
    
- **优雅停机配置**：`graceful-stop`，HAProxy 热重载 / 停机时，旧 Worker 进程会等待现有连接处理完成后再退出，默认启用，生产环境需保持开启

#### 2. 无损上线机制

- **slowstart 慢启动**：节点上线后，权重在指定时间内线性增长，逐步接收流量，避免节点刚启动就接收全量流量导致过载，生产环境必配
    
    - 配置示例：`server web01 192.168.1.101:80 check slowstart 60s`
    
- **健康检查预验证**：节点上线前，必须通过连续`rise`次健康检查，才会标记为 UP，避免未就绪的节点接收流量
- **权重逐步调整**：运行时通过 stats socket 逐步调整节点权重，从 0 逐步增加到 100%，实现完全无损的流量接入

## 4.5 HAProxy 集群高可用架构

单节点 HAProxy 存在单点故障风险，生产环境必须部署 HAProxy 集群，通过 VRRP 协议实现 VIP 漂移，解决自身单点故障问题，同时配合会话状态同步，实现主备切换的无感知。

### 4.5.1 集群高可用核心架构设计

HAProxy 集群高可用架构基于「**Keepalived+VRRP 协议实现 VIP 漂移** + **HAProxy 双节点主备 / 双主部署** + **会话状态集群同步**」三大核心模块构建，核心设计目标：

1. 主节点故障时，VIP 自动漂移到备节点，实现秒级故障切换，客户端无感知
2. 主备切换后，会话保持规则不失效，用户业务无中断
3. 具备脑裂防护能力，避免双主场景导致的网络异常
4. 主备节点配置完全一致，支持配置自动同步

### 4.5.2 主备高可用架构（标准生产架构）

主备架构是生产环境最常用的集群架构，两个 HAProxy 节点一主一备，同一时间只有主节点持有 VIP 并处理流量，备节点处于热备状态，主节点故障时自动接管 VIP 与流量。

#### 1. 核心架构原理

- 主备节点均运行 HAProxy 与 Keepalived 服务
- Keepalived 通过 VRRP 协议在二层网络中广播节点状态，选举主节点
- 主节点定期发送 VRRP 广播报文，备节点监听报文，当超过 3 个报文周期未收到主节点广播时，备节点自动升级为主节点，接管 VIP
- Keepalived 配置 HAProxy 进程健康检查，当主节点的 HAProxy 进程故障时，自动触发主备切换，避免 HAProxy 进程宕机但服务器正常的场景

#### 2. 生产级完整配置示例

##### （1）主节点 Keepalived 配置（/etc/keepalived/keepalived.conf）

plaintext

```
! 全局配置
global_defs {
    router_id haproxy_master
    script_user root
    enable_script_security
}

! HAProxy进程健康检查脚本
vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight -20
    fall 3
    rise 2
}

! VRRP实例配置
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    # 开启非抢占模式，主节点恢复后不抢回主角色，避免二次切换
    nopreempt
    # 认证配置，防止非法节点加入集群
    authentication {
        auth_type PASS
        auth_pass haproxy@2026
    }
    # 虚拟VIP地址
    virtual_ipaddress {
        192.168.1.200/24 dev eth0
    }
    # 绑定健康检查脚本
    track_script {
        chk_haproxy
    }
    # 切换通知脚本，用于日志告警与配置同步
    notify_master /etc/keepalived/scripts/notify_master.sh
    notify_backup /etc/keepalived/scripts/notify_backup.sh
    notify_fault /etc/keepalived/scripts/notify_fault.sh
}
```

##### （2）备节点 Keepalived 配置

plaintext

```
! 全局配置
global_defs {
    router_id haproxy_backup
    script_user root
    enable_script_security
}

! HAProxy进程健康检查脚本
vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight -20
    fall 3
    rise 2
}

! VRRP实例配置
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 80
    advert_int 1
    nopreempt
    authentication {
        auth_type PASS
        auth_pass haproxy@2026
    }
    virtual_ipaddress {
        192.168.1.200/24 dev eth0
    }
    track_script {
        chk_haproxy
    }
    notify_master /etc/keepalived/scripts/notify_master.sh
    notify_backup /etc/keepalived/scripts/notify_backup.sh
    notify_fault /etc/keepalived/scripts/notify_fault.sh
}
```

##### （3）HAProxy 配置

主备节点的 HAProxy 配置完全一致，包括前端、后端、健康检查、会话保持等所有配置，确保切换后转发规则完全一致。

#### 3. 脑裂防护机制

脑裂是 VRRP 集群的核心风险，指主备节点同时认为自己是主节点，同时持有 VIP，导致网络异常。生产环境必须配置以下防护措施：

1. **双链路检测**：主备节点通过业务网 + 管理网双链路通信，避免单链路故障导致的脑裂
2. **仲裁节点检测**：配置第三方仲裁节点（如网关、DNS 服务器），当节点无法收到对端 VRRP 报文时，先检测仲裁节点的连通性，无法连通则认为自身网络故障，不触发主备切换
3. **优先级动态调整**：通过健康检查脚本动态调整节点优先级，当 HAProxy 进程故障时，降低优先级，触发主备切换
4. **非抢占模式**：启用`nopreempt`非抢占模式，主节点恢复后不抢回主角色，避免二次切换与双主冲突
5. **防火墙规则**：严格限制 VRRP 报文的源地址，仅允许主备节点之间的 VRRP 通信，防止非法节点干扰

### 4.5.3 双主（负载分担）高可用架构

双主架构适用于流量较大的场景，两个 HAProxy 节点同时处于主节点状态，各自持有一个 VIP，同时处理流量，互相作为对方的备份节点，充分利用服务器资源。

- **核心架构原理**：配置两个 VRRP 实例，节点 A 在实例 1 中为主节点，在实例 2 中为备节点；节点 B 在实例 1 中为备节点，在实例 2 中为主节点；两个 VIP 分别绑定到两个实例，业务流量通过 DNS 轮询分发到两个 VIP，实现负载分担
- **适用场景**：高并发业务场景，单节点 HAProxy 无法承载全量流量，需要充分利用两台服务器的性能
- **核心优势**：资源利用率高，两台服务器同时工作，无资源浪费；单个节点故障时，另一个节点接管两个 VIP，保证业务不中断

### 4.5.4 集群会话状态同步机制

主备切换后，若会话数据仅存储在原主节点的本地粘性表中，会导致会话保持失效，HAProxy 通过`peers`段实现集群节点间的粘性表数据实时同步，确保主备切换后会话完全一致。

#### 1. 核心语法与配置

plaintext

```
# 全局peers段，定义集群节点
peers haproxy_cluster
    # 本地节点，必须与主机名一致
    peer haproxy01 192.168.1.100:1024
    # 对端节点
    peer haproxy02 192.168.1.101:1024

# 后端配置段，启用粘性表同步
backend web_back
    mode http
    balance roundrobin
    # 粘性表绑定到peers集群，实现数据同步
    stick-table type string size 100k expire 30m peers haproxy_cluster
    stick on cookie(SESSIONID)
    server web01 192.168.1.101:80 check
    server web02 192.168.1.102:80 check
```

#### 2. 核心执行规则

1. 主备节点之间通过 TCP 1024 端口建立同步连接，实时同步粘性表的新增、更新、过期数据
2. 节点启动时，自动从对端节点同步全量粘性表数据，实现冷启动后的会话恢复
3. 同步数据仅包含会话元数据，无业务数据，性能开销极低，不影响流量转发
4. 主备切换后，新主节点已拥有全量会话数据，会话保持规则完全生效，用户无感知

## 4.6 生产级高可用完整配置模板

### 4.6.1 HTTP Web 集群高可用配置

plaintext

```
# 全局配置段
global
    daemon
    master-worker
    nbthread 4
    maxconn 100000
    log /dev/log local0 info
    stats socket /var/run/haproxy.sock mode 660 level admin
    pidfile /var/run/haproxy.pid
    user haproxy
    group haproxy
    ssl-default-bind-options no-sslv3 no-tlsv10 no-tlsv11
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
    # 开启外部脚本检查（可选）
    external-check
    external-check path /etc/haproxy/scripts/

# 集群会话同步配置
peers haproxy_cluster
    peer haproxy01 192.168.1.100:1024
    peer haproxy02 192.168.1.101:1024

# 默认配置段
defaults
    mode http
    log global
    option httplog
    option forwardfor
    option http-server-close
    # 核心高可用配置
    option redispatch
    retries 3
    retry-on conn-failure timeout 500 empty-response invalid-response
    # 超时配置
    timeout connect 5s
    timeout client 30s
    timeout server 30s
    timeout http-request 10s
    timeout http-keep-alive 5s
    timeout queue 30s
    timeout check 2s
    # 负载均衡默认配置
    balance roundrobin
    # 健康检查默认配置
    inter 2s
    rise 2
    fall 3

# 前端入口
frontend https_front
    bind 0.0.0.0:443 ssl crt /etc/haproxy/certs/www.example.com.pem alpn h2,http/1.1
    mode http
    # 容灾ACL配置
    acl main_down nbsrv(www_back) lt 2
    acl backup_available nbsrv(www_backup) ge 1
    use_backend www_backup if main_down backup_available
    default_backend www_back

# 主后端集群
backend www_back
    mode http
    # 七层健康检查
    option httpchk GET /health HTTP/1.1
    http-check send hdr Host www.example.com
    http-check expect status 200
    # 被动健康检查与熔断
    observe layer7
    on-error fail-check
    error-limit 5
    on-marked-down shutdown-sessions
    # Cookie会话保持
    cookie SERVERID insert nocache indirect secure httponly maxidle 3600s
    # 服务器节点，慢启动60秒
    server web01 192.168.1.101:80 check cookie web01 slowstart 60s
    server web02 192.168.1.102:80 check cookie web02 slowstart 60s
    server web03 192.168.1.103:80 check cookie web03 slowstart 60s

# 备份后端集群
backend www_backup
    mode http
    option httpchk GET /health HTTP/1.1
    http-check send hdr Host www.example.com
    http-check expect status 200
    server backup01 192.168.2.101:80 check
    server backup02 192.168.2.102:80 check
```

### 4.6.2 MySQL 主备 TCP 集群高可用配置

plaintext

```
# 全局配置段
global
    daemon
    master-worker
    nbthread 2
    maxconn 50000
    log /dev/log local0 info
    stats socket /var/run/haproxy.sock mode 660 level admin
    pidfile /var/run/haproxy.pid
    user haproxy
    group haproxy

# 集群会话同步
peers mysql_cluster
    peer haproxy01 192.168.1.100:1024
    peer haproxy02 192.168.1.101:1024

# 默认配置段
defaults
    mode tcp
    log global
    option tcplog
    option redispatch
    retries 2
    timeout connect 3s
    timeout client 1800s
    timeout server 1800s
    timeout check 1s
    balance leastconn
    inter 1s
    rise 2
    fall 3

# MySQL主库监听段
listen mysql_master
    bind 0.0.0.0:3306
    mode tcp
    # TCP健康检查
    option tcp-check
    tcp-check expect rstring "5\\.[5-7]\\.[0-9]+"
    # 被动健康检查
    observe layer4
    on-error fail-check
    # 粘性表会话保持，集群同步
    stick-table type ip size 10k expire 30m peers mysql_cluster
    stick on src
    # 主库节点，故障时切换到备库
    server mysql_master 192.168.1.201:3306 check
    server mysql_slave 192.168.1.202:3306 check backup
```

继续下一章