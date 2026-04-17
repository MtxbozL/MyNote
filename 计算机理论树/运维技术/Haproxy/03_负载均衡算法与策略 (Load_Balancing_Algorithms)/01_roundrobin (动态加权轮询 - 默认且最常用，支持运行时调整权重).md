# 第二章 HAProxy 核心配置体系与语法规范

本章为 HAProxy 实操学习的核心基础，完整定义 HAProxy 配置文件的语法规则、分段架构、作用域体系与生效机制，所有内容均基于 2.8.x LTS 版本规范，严格遵循 HAProxy 官方词法与语法标准，无冗余表述与非标准化配置。

## 2.1 配置文件整体架构与生命周期

HAProxy 采用**分段式声明配置架构**，配置文件为纯文本格式，无编译环节，所有配置均通过声明式指令完成，核心设计逻辑为「控制平面全局约束 - 默认规则继承 - 流量入口定义 - 后端服务池定义」的四层分层架构，彻底分离流量入口与后端服务池的配置职责，实现高内聚低耦合的配置管理。

### 2.1.1 配置文件核心分段体系

HAProxy 配置文件分为**核心必选段**与**辅助可选段**两大类，所有段的作用域、生命周期与执行优先级严格区分，无交叉重叠，具体分类如下：

表格

|分段类型|分段名称|核心职责|必选性|作用域|
|---|---|---|---|---|
|核心必选段|`global`|全局配置段，定义整个 HAProxy 实例的进程、性能、安全、日志等底层约束|必选|实例全生命周期，所有其他分段均受其约束|
|核心必选段|`defaults`|默认配置段，定义前端、后端、监听段的通用默认规则，支持配置继承与覆盖|推荐必选|后续匹配的前端、后端、监听段，支持多 defaults 分段层级继承|
|核心必选段|`frontend`|前端配置段，定义客户端流量入口、监听地址 / 端口、流量接入规则与路由分发逻辑|必选（流量入口）|客户端到 HAProxy 的接入层，仅处理入站流量|
|核心必选段|`backend`|后端配置段，定义上游服务器池、负载均衡算法、健康检查、后端转发规则|必选（流量转发目标）|HAProxy 到上游服务器的转发层，仅处理出站流量|
|核心可选段|`listen`|监听配置段，`frontend`与`backend`的合一简化段，同时定义流量入口与后端服务池|可选|完整的流量接入 - 转发全链路，适用于 TCP 服务与简单 HTTP 服务|
|辅助可选段|`userlist`|用户列表段，定义 HTTP 基本认证、TCP 认证的用户与权限规则|可选|全局认证规则库，可被所有前端 / 后端段引用|
|辅助可选段|`peers`|集群对等段，定义 HAProxy 集群节点间的状态同步规则，用于会话状态、stick-table 数据同步|可选|集群节点间的数据同步域|
|辅助可选段|`mailers`|邮件告警段，定义邮件服务器信息，用于健康检查状态变更、故障事件的邮件告警|可选|全局告警规则库，可被所有后端段引用|
|辅助可选段|`resolvers`|域名解析段，定义 DNS 解析服务器与规则，用于后端服务器域名动态解析|可选|全局 DNS 解析规则库，可被所有后端段引用|

### 2.1.2 配置加载与执行生命周期

HAProxy 配置的加载与执行严格遵循「解析 - 校验 - 继承 - 生效」的线性流程，与第一章的 Master-Worker 架构完全对应，完整生命周期如下：

1. **配置解析阶段**：Master 进程启动 / 接收重载信号时，按配置文件的书写顺序逐行解析，先解析`global`段，再按顺序解析后续所有分段，解析过程中完成词法与语法校验
2. **继承合并阶段**：Master 进程完成基础解析后，按继承规则将`defaults`段的配置合并到对应的`frontend`/`backend`/`listen`段，生成每个分段的完整生效配置
3. **合法性校验阶段**：Master 进程对合并后的完整配置进行全量合法性校验，包括端口占用、证书有效性、配置逻辑一致性、资源限制合规性，校验失败则终止加载，保留原有运行配置
4. **配置生效阶段**：校验通过后，Master 进程将完整配置下发给所有 Worker 进程，Worker 进程基于新配置启动事件循环，接管流量处理；热重载场景下，旧 Worker 进程完成现有连接处理后优雅退出，实现零停机配置更新

## 2.2 核心语法规范与词法规则

HAProxy 配置语法采用严格的声明式格式，无脚本化逻辑，所有指令均遵循统一的词法规则，大小写敏感，格式错误会直接导致配置校验失败，本节定义所有配置的通用语法规范。

### 2.2.1 基础语法格式

1. **指令行格式**：每一行配置为一条独立指令，通用格式为：
    
    plaintext
    
    ```
    指令名称 [参数1] [参数2] ... [参数N]
    ```
    
    - 指令名称为 HAProxy 官方预定义关键字，**大小写敏感**，所有核心指令均为小写
    - 指令与参数、参数与参数之间通过**一个或多个空格 / 制表符**分隔，无固定列宽限制
    - 单条指令行最大长度无硬性限制，推荐单行不超过 256 个字符，超长配置可通过反斜杠`\`换行
    
2. **注释规则**：
    
    - 行注释：以`#`开头的行，整行均为注释，不参与配置解析
    - 行内注释：指令行末尾以`#`开头的内容为行内注释，`#`前的配置内容正常解析
    - 注释不可嵌套，不可出现在指令名称与核心参数之间
    
3. **换行与续行规则**：
    
    - 单条指令可通过反斜杠`\`实现跨行书写，`\`必须为行尾最后一个字符，后续行的前导空格会被忽略
    - 空行会被解析器直接忽略，无任何语义影响，可用于配置分段排版
    

### 2.2.2 数据类型与单位规范

HAProxy 配置中所有参数均有严格的数据类型与单位规范，无隐式类型转换，类型不匹配会直接导致配置校验失败。

1. **数值类型**
    
    - 整数：无符号整数，用于连接数、权重、重试次数、阈值等配置，取值范围根据指令定义有明确约束
    - 布尔值：仅支持`on`/`off`、`enabled`/`disabled`、`yes`/`no`三组取值，大小写敏感，不可使用 1/0 替代
    
2. **时间单位规范**
    
    所有超时、间隔类配置均支持统一的时间单位，单位为小写字母，不可省略，无默认单位，完整单位体系如下：
    
    表格
    
    |单位符号|全称|换算关系|适用场景|
    |---|---|---|---|
    |`us`|微秒|1us = 10^-6 秒|高精度超时、内核级 TCP 参数配置|
    |`ms`|毫秒|1ms = 10^-3 秒|健康检查超时、高精度请求超时|
    |`s`|秒|基础时间单位|通用超时、健康检查间隔配置|
    |`m`|分钟|1m = 60s|长连接超时、会话保持时长|
    |`h`|小时|1h = 60m = 3600s|日志轮转、长会话超时|
    |`d`|天|1d = 24h = 86400s|证书有效期校验、持久化会话时长|
    
    - 示例：`timeout client 30s`、`inter 200ms`、`timeout http-keep-alive 5s`
    
3. **存储容量单位规范**
    
    所有缓冲区、文件大小、限流阈值类配置均支持统一的容量单位，单位为大小写敏感，完整单位体系如下：
    
    表格
    
    |单位符号|换算关系|适用场景|
    |---|---|---|
    |`B`|字节，基础单位|缓冲区最小尺寸配置|
    |`k`|千字节，1k = 1024B|通用缓冲区、请求体限制配置|
    |`M`|兆字节，1M = 1024k|大文件传输、响应体限制配置|
    |`G`|吉字节，1G = 1024M|日志文件、超大传输场景配置|
    
    - 示例：`tune.bufsize 16k`、`maxrewrite 1k`
    
4. **网络地址规范**
    
    - IPv4 地址：标准点分十进制格式，支持`0.0.0.0`全地址监听，支持 CIDR 格式网段，如`192.168.1.0/24`
    - IPv6 地址：标准冒分十六进制格式，需用方括号包裹，如`[::]:80`、`[2001:db8::1]:443`
    - 端口范围：支持单端口`80`、连续端口范围`8000-8080`，端口取值范围 1-65535
    - 主机名：支持标准域名格式，需配合`resolvers`段实现动态解析
    

### 2.2.3 命名规范

所有自定义命名的分段、ACL、规则、服务器均遵循统一的命名规范，不符合规范的命名会导致配置解析失败：

1. 命名仅支持**大小写字母、数字、下划线`_`、连字符`-`、点`.`**，不可包含空格、特殊符号与中文
2. 命名首字符必须为字母，不可为数字、连字符与点
3. 命名长度不超过 63 个字符，区分大小写
4. 同一类型的命名全局唯一，不可重复：如不可存在两个同名的`frontend`段，不可存在同一`backend`下两个同名的`server`
5. 禁止使用 HAProxy 预定义关键字作为命名，如`balance`、`mode`、`timeout`等

## 2.3 核心配置段详解

本节为第二章核心内容，完整定义每个核心配置段的所有关键指令、技术原理、取值规范与约束条件，无冗余非核心指令，所有内容均为生产环境与学习必备知识点。

### 2.3.1 `global` 全局配置段

`global`段为实例级唯一配置段，定义整个 HAProxy 实例的底层运行约束，所有配置均为实例全局生效，不可被其他分段覆盖，核心分为 6 大类配置模块。

#### 模块 1：进程与线程管理配置

对应第一章的 Master-Worker 架构，定义实例的进程、线程调度规则，核心指令如下：

表格

|指令|语法格式|核心作用|取值规范与约束|
|---|---|---|---|
|`daemon`|`daemon`|配置 HAProxy 以守护进程模式后台运行|无参数，生产环境必选配置|
|`nbproc`|`nbproc <数量>`|配置 Worker 进程数量，每个进程为独立的调度单元|整数，推荐与物理 CPU 核心数 1:1 配置，最大不超过 64；NUMA 架构推荐使用，通用场景优先使用`nbthread`|
|`nbthread`|`nbthread <数量>`|配置单个 Worker 进程内的线程数量，共享文件描述符，无锁竞争|整数，推荐与物理 CPU 核心数 1:1 配置，最大不超过 64；通用多核服务器首选配置，与`nbproc`互斥，不可同时配置|
|`cpu-map`|`cpu-map <进程/线程ID> <CPU核心编号>`|将指定进程 / 线程绑定到固定 CPU 核心，避免上下文切换开销|核心编号从 0 开始，需与`nbproc`/`nbthread`配置对应，生产环境性能优化必选配置|
|`master-worker`|`master-worker`|启用 Master-Worker 主从架构模式|无参数，2.0 + 版本默认启用，不可关闭|
|`maxconn`|`maxconn <数值>`|全局最大并发连接数限制，单个 Worker 进程的最大连接数为该值 / Worker 数量|整数，不可超过系统 ulimit 文件描述符限制，生产环境需根据服务器配置调整|
|`ulimit-n`|`ulimit-n <数值>`|强制设置 HAProxy 进程的最大文件描述符数量|整数，必须大于`maxconn`的 2 倍（每个连接占用 2 个文件描述符），推荐自动计算，无需手动配置|

#### 模块 2：性能调优配置

定义 HAProxy 的 I/O 模型、缓冲区、TCP 协议优化等底层性能参数，核心指令如下：

表格

|指令|语法格式|核心作用|约束条件|
|---|---|---|---|
|`tune.bufsize`|`tune.bufsize <容量>`|设置每个连接的读写缓冲区大小|默认 16k，最大不超过 64k，过大会导致内存占用过高，过小会导致大请求处理失败|
|`tune.maxaccept`|`tune.maxaccept <数值>`|设置单个 Worker 进程一次事件循环可接受的最大 TCP 连接数|默认值为内核自动调整，高并发场景可适当调大，避免连接饥饿|
|`tune.tcp.sack`|`tune.tcp.sack <布尔值>`|启用 / 禁用 TCP 选择性确认机制|默认启用，高丢包网络环境必选开启|
|`tune.ssl.default-dh-param`|`tune.ssl.default-dh-param <位数>`|设置 SSL/TLS 的 DH 参数位数|推荐 2048，最低不小于 1024，2048 为安全基线|

#### 模块 3：日志与统计配置

定义全局日志输出、统计接口、运维管控能力，核心指令如下：

表格

|指令|语法格式|核心作用|约束条件|
|---|---|---|---|
|`log`|`log <地址> <设施> <日志级别>`|配置全局日志服务器，定义日志输出目标|支持 syslog 协议，地址可为本地`/dev/log`或远程 syslog 服务器 IP: 端口；设施为 syslog 标准设施，如`local0`；日志级别从高到低为`emerg`/`alert`/`crit`/`err`/`warning`/`notice`/`info`/`debug`|
|`stats socket`|`stats socket <路径> [权限参数]`|配置 HAProxy 的 CLI 运维管控套接字，支持运行时状态查询、配置调整|路径为 Unix 套接字文件路径，推荐配置`mode 660 level admin`，实现管理员级运维权限|
|`stats timeout`|`stats timeout <时间>`|设置统计接口的连接超时时间|推荐 30s，避免无效连接占用资源|

#### 模块 4：安全与权限配置

定义进程权限、安全隔离、攻击防护等全局安全规则，核心指令如下：

表格

|指令|语法格式|核心作用|约束条件|
|---|---|---|---|
|`chroot`|`chroot <目录路径>`|将 HAProxy 进程限制在指定目录的 chroot 环境中，实现文件系统隔离|目录必须为空，无执行权限，生产环境安全加固必选配置|
|`user`/`group`|`user <用户名>` / `group <组名>`|设置 Worker 进程的运行用户与用户组，去除 root 权限|必须为系统非特权用户，生产环境必选配置，避免权限溢出|
|`pidfile`|`pidfile <文件路径>`|设置 HAProxy Master 进程的 PID 文件路径|用于进程管理、优雅重启，推荐`/var/run/haproxy.pid`|
|`set-dumpable`|`set-dumpable <布尔值>`|启用 / 禁用进程核心转储，默认禁用|调试场景开启，生产环境必须禁用，避免内存数据泄露|

#### 模块 5：SSL/TLS 全局配置

定义全局 SSL/TLS 协议、加密套件、证书管理规则，核心指令如下：

表格

|指令|语法格式|核心作用|约束条件|
|---|---|---|---|
|`ssl-default-bind-ciphers`|`ssl-default-bind-ciphers <加密套件列表>`|配置前端监听端口的默认 TLS 加密套件|遵循 OpenSSL 加密套件规范，生产环境需禁用不安全套件，仅启用 TLS1.2 + 兼容套件|
|`ssl-default-bind-options`|`ssl-default-bind-options <参数>`|配置前端监听端口的默认 TLS 协议选项|生产环境必选配置`no-sslv3 no-tlsv10 no-tlsv11`，禁用不安全的 SSL/TLS 版本|
|`ssl-default-server-ciphers`|`ssl-default-server-ciphers <加密套件列表>`|配置后端服务器连接的默认 TLS 加密套件|与前端加密套件规范一致，用于后端 HTTPS 服务的 TLS 握手|
|`ssl-default-server-options`|`ssl-default-server-options <参数>`|配置后端服务器连接的默认 TLS 协议选项|同前端安全规范，禁用低版本 TLS 协议|

### 2.3.2 `defaults` 默认配置段

`defaults`段用于定义`frontend`/`backend`/`listen`段的通用默认配置，实现配置复用，避免重复代码，支持多`defaults`段层级定义，遵循「就近继承、显式覆盖」的核心规则。

#### 2.3.2.1 继承规则

1. 匿名`defaults`段：无命名的`defaults`段，其配置会被后续所有未显式指定默认段的`frontend`/`backend`/`listen`段继承
2. 命名`defaults`段：格式为`defaults <名称>`，仅被显式通过`use_defaults <名称>`指令引用的分段继承
3. 覆盖规则：子分段中显式配置的指令，会完全覆盖从`defaults`段继承的同名指令；未显式配置的指令，完全继承`defaults`段的配置
4. 多`defaults`段规则：后续的匿名`defaults`段会完全覆盖之前的匿名`defaults`段配置，无合并逻辑

#### 2.3.2.2 核心配置模块

`defaults`段的核心配置分为 7 大类，所有配置均可被`frontend`/`backend`/`listen`段继承与覆盖，为生产环境配置的核心复用模块。

##### 模块 1：代理模式配置

定义代理的工作层级，是 4 层 TCP 代理还是 7 层 HTTP 代理，核心指令：

表格

|指令|语法格式|核心作用|约束条件|
|---|---|---|---|
|`mode`|`mode <tcp/http>`|定义代理的工作模式，决定整个流量处理的 OSI 层级|必选配置，`tcp`为 4 层传输层代理，`http`为 7 层应用层代理；同一流量链路的`frontend`与`backend`必须为同一 mode|

##### 模块 2：超时配置（重中之重）

HAProxy 的超时配置直接决定连接生命周期、资源释放、攻击防护能力，所有超时参数均有明确的生命周期边界，无模糊语义，核心指令如下：

表格

|指令|语法格式|核心作用|生命周期边界|推荐取值|
|---|---|---|---|---|
|`timeout connect`|`timeout connect <时间>`|HAProxy 与后端服务器建立 TCP 连接的超时时间，即 TCP 三次握手的最大耗时|从发送 SYN 包到收到 ACK+SYN 包的完整握手过程|3s-5s，生产环境不超过 10s|
|`timeout client`|`timeout client <时间>`|客户端与 HAProxy 之间的 TCP 连接空闲超时时间，即客户端无数据发送的最大空闲时长|客户端连接的全生命周期，空闲计时在收到客户端数据后重置|30s-1m，长连接场景可适当延长|
|`timeout server`|`timeout server <时间>`|HAProxy 与后端服务器之间的 TCP 连接空闲超时时间，即后端服务器无数据返回的最大空闲时长|后端连接的全生命周期，空闲计时在收到后端数据后重置|30s-1m，慢接口场景可适当延长|
|`timeout http-request`|`timeout http-request <时间>`|客户端发送完整 HTTP 请求的最大超时时间，从 TCP 连接建立到请求头 / 请求体完全接收完成|仅 HTTP 模式生效，从连接建立到请求完全接收，防止慢连接攻击|5s-10s，生产环境必选配置|
|`timeout http-keep-alive`|`timeout http-keep-alive <时间>`|HTTP Keep-Alive 模式下，连接空闲等待下一个请求的最大时长|仅 HTTP 模式生效，两个连续 HTTP 请求之间的空闲间隔|1s-5s，平衡性能与资源占用|
|`timeout queue`|`timeout queue <时间>`|请求在后端服务器队列中等待的最大超时时间，当后端服务器`maxconn`满时生效|从请求进入队列到被分配到后端服务器的等待时长|10s-30s，避免请求长时间排队|
|`timeout check`|`timeout check <时间>`|后端服务器健康检查的超时时间，超过该时间未收到响应则判定检查失败|单次健康检查的完整生命周期，独立于`timeout connect`|1s-2s，需小于`inter`健康检查间隔|

##### 模块 3：日志配置

定义默认的日志输出规则，核心指令：

表格

|指令|语法格式|核心作用|约束条件|
|---|---|---|---|
|`log`|`log global`|继承`global`段的全局日志配置|生产环境推荐配置，避免重复定义日志服务器|
|`option httplog`|`option httplog`|启用 HTTP 模式的详细日志，记录请求方法、URL、状态码、响应时长等完整 HTTP 信息|仅 HTTP 模式生效，生产环境 HTTP 场景必选配置|
|`option tcplog`|`option tcplog`|启用 TCP 模式的详细日志，记录连接时长、字节数、连接状态等 TCP 信息|仅 TCP 模式生效，生产环境 TCP 场景必选配置|

##### 模块 4：负载均衡默认配置

定义默认的负载均衡算法与调度规则，核心指令：

表格

|指令|语法格式|核心作用|约束条件|
|---|---|---|---|
|`balance`|`balance <算法名称> [算法参数]`|定义默认的负载均衡调度算法|可被`backend`段显式覆盖，HTTP 模式默认`roundrobin`，TCP 模式默认`roundrobin`|
|`retries`|`retries <次数>`|定义后端连接失败后的最大重试次数|推荐 2-3 次，避免过度重试导致后端压力过大|
|`option redispatch`|`option redispatch`|启用会话重分发机制，当会话保持的后端服务器故障时，自动将请求分发到其他健康服务器|仅 HTTP 模式生效，生产环境必选配置，避免单点故障导致的请求失败|

##### 模块 5：健康检查默认配置

定义后端服务器健康检查的通用规则，核心指令：

表格

|指令|语法格式|核心作用|约束条件|
|---|---|---|---|
|`option httpchk`|`option httpchk <方法> <URL> <HTTP版本>`|启用 HTTP 模式的 7 层健康检查，通过 HTTP 请求状态码判定服务器健康状态|仅 HTTP 模式生效，默认发送`OPTIONS / HTTP/1.0`请求，推荐显式配置检查路径与方法|
|`option tcp-check`|`option tcp-check`|启用 TCP 模式的 4 层健康检查，通过 TCP 端口连通性判定服务器健康状态|仅 TCP 模式生效，为 TCP 模式默认健康检查方式|
|`inter`|`inter <时间>`|定义健康检查的默认间隔时间|推荐 2s，生产环境根据业务场景调整|
|`rise`|`rise <次数>`|定义故障服务器恢复上线所需的连续健康检查成功次数|推荐 2-3 次，避免抖动导致的频繁上下线|
|`fall`|`fall <次数>`|定义健康服务器标记为故障下线所需的连续健康检查失败次数|推荐 3 次，平衡故障感知速度与稳定性|

##### 模块 6：HTTP 协议优化配置

仅 HTTP 模式生效，定义 HTTP 请求处理的通用规则，核心指令：

表格

|指令|语法格式|核心作用|约束条件|
|---|---|---|---|
|`option forwardfor`|`option forwardfor [except <网段>]`|自动在 HTTP 请求头中添加`X-Forwarded-For`字段，传递客户端真实 IP 给后端服务器|生产环境 HTTP 场景必选配置，`except`参数用于排除信任的前置代理网段|
|`option http-server-close`|`option http-server-close`|启用服务端连接关闭模式，客户端与 HAProxy 之间保持 Keep-Alive，HAProxy 与后端服务器之间短连接|兼容绝大多数后端服务，平衡性能与后端连接资源占用，生产环境推荐配置|
|`option http-ignore-proxy`|`option http-ignore-proxy`|忽略 HTTP 请求中的代理相关头信息，防止代理请求滥用|生产环境必选配置，提升安全性|

### 2.3.3 `frontend` 前端配置段

`frontend`段定义客户端流量的接入入口，负责监听客户端请求、解析流量特征、执行路由规则，将请求分发到对应的`backend`段，是流量处理的第一环节。

#### 核心指令体系

表格

|指令分类|核心指令|语法格式与核心作用|约束条件|
|---|---|---|---|
|基础配置|`bind`|语法：`bind <监听地址>:<端口> [参数1] [参数2] ...`<br><br>核心作用：定义前端监听的 IP 地址与端口，是`frontend`段的必选指令，可配置多条`bind`指令监听多个端口|地址为`0.0.0.0`表示监听所有 IPv4 地址，`[::]`表示监听所有 IPv6 地址；支持 SSL/TLS、TCP 参数、PROXY 协议等扩展参数|
|基础配置|`mode`|语法：`mode <tcp/http>`<br><br>核心作用：定义前端的代理工作模式，必须与目标`backend`段的 mode 一致|必选配置，未配置则继承`defaults`段的 mode|
|基础配置|`default_backend`|语法：`default_backend <backend名称>`<br><br>核心作用：定义默认后端服务池，当所有路由规则均未匹配时，请求将被分发到该 backend|必选配置，不可指向不存在的 backend 段|
|路由配置|`use_backend`|语法：`use_backend <backend名称> if <ACL条件>`<br><br>核心作用：定义基于 ACL 规则的流量路由，当 ACL 条件匹配时，请求被分发到指定 backend|可配置多条，按书写顺序从上到下匹配，匹配到第一条后终止匹配|
|接入控制|`maxconn`|语法：`maxconn <数值>`<br><br>核心作用：定义该前端入口的最大并发连接数限制|不可超过 global 段的全局 maxconn 限制|
|接入控制|`acl`|语法：`acl <ACL名称> <匹配规则> <匹配值>`<br><br>核心作用：定义访问控制列表，用于流量特征匹配，是路由、限流、访问控制的核心基础|ACL 名称遵循命名规范，匹配规则为 HAProxy 预定义的匹配因子，支持数百种匹配维度|
|HTTP 配置|`http-request`|语法：`http-request <动作> [条件]`<br><br>核心作用：定义 HTTP 请求的处理规则，支持请求头增删改、URL 重写、重定向、限流、拒绝等动作|仅 HTTP 模式生效，按书写顺序从上到下执行|
|HTTP 配置|`http-response`|语法：`http-response <动作> [条件]`<br><br>核心作用：定义 HTTP 响应的处理规则，支持响应头增删改、内容过滤、状态码修改等动作|仅 HTTP 模式生效，按书写顺序从上到下执行|

#### `bind` 指令核心扩展参数

`bind`指令是前端配置的核心，其扩展参数直接决定监听端口的协议、安全、性能特性，生产环境必备核心参数如下：

- `ssl`：启用该端口的 SSL/TLS 加密，必须配合`crt`参数指定证书文件
- `crt <证书路径>`：指定 SSL/TLS 证书文件路径，支持 PEM 格式证书，可配置多个证书实现 SNI 多域名支持
- `alpn <协议列表>`：指定 TLS ALPN 扩展的协议列表，用于 HTTP/2 协商，推荐配置`alpn h2,http/1.1`
- `accept-proxy`：启用 PROXY 协议接收，用于前置代理传递客户端真实 IP，适用于 LVS+HAProxy 架构
- `backlog <数值>`：指定 TCP 监听的 backlog 队列大小，需与内核参数`net.core.somaxconn`匹配，高并发场景推荐配置 1024 以上
- `tcp-keepalive`：启用 TCP 保活机制，检测空闲客户端连接的有效性

### 2.3.4 `backend` 后端配置段

`backend`段定义上游服务器池，负责负载均衡调度、健康检查、后端连接管理、请求转发规则，是流量处理的最终转发环节。

#### 核心指令体系

表格

|指令分类|核心指令|语法格式与核心作用|约束条件|
|---|---|---|---|
|基础配置|`mode`|语法：`mode <tcp/http>`<br><br>核心作用：定义后端的代理工作模式，必须与对应 frontend 段的 mode 一致|必选配置，未配置则继承 defaults 段的 mode|
|负载均衡|`balance`|语法：`balance <算法名称> [算法参数]`<br><br>核心作用：定义该后端服务器池的负载均衡调度算法，是后端配置的核心指令|未配置则继承 defaults 段的 balance 配置|
|服务器定义|`server`|语法：`server <服务器名称> <地址>:<端口> [参数1] [参数2] ...`<br><br>核心作用：定义上游服务器节点，可配置多条，一个 backend 段可包含多个服务器节点|服务器名称遵循命名规范，地址可为 IP 或域名，支持健康检查、权重、连接限制等数十种扩展参数|
|健康检查|`option httpchk`/`option tcp-check`|语法同 defaults 段，定义该后端的健康检查方式，覆盖 defaults 段的配置|httpchk 仅 HTTP 模式生效，tcp-check 仅 TCP 模式生效|
|会话保持|`cookie`|语法：`cookie <Cookie名称> <参数>`<br><br>核心作用：基于 Cookie 实现 HTTP 会话保持，确保同一客户端的请求始终转发到同一后端服务器|仅 HTTP 模式生效，需配合 server 指令的`cookie`参数使用|
|会话保持|`stick-table`|语法：`stick-table type <类型> size <容量> [expire <时间>]`<br><br>核心作用：定义粘性会话表，基于源 IP、请求头等特征实现会话保持，支持 TCP 与 HTTP 模式|需配合`stick on`指令使用，支持集群间数据同步|
|重试机制|`retries`/`option redispatch`|语法同 defaults 段，定义该后端的请求重试与重分发规则，覆盖 defaults 段的配置|无特殊需求推荐继承 defaults 段配置|
|HTTP 处理|`http-request`/`http-response`|语法同 frontend 段，定义转发到后端前的请求处理规则与返回客户端前的响应处理规则|仅 HTTP 模式生效，执行顺序晚于 frontend 段的同名指令|

#### 核心负载均衡算法详解

HAProxy 内置十余种负载均衡算法，所有算法均为无锁化实现，调度性能无显著差异，核心算法的技术原理、适用场景如下：

表格

|算法名称|语法|技术原理|核心优势|核心局限性|最佳适用场景|
|---|---|---|---|---|---|
|轮询算法|`balance roundrobin`|动态加权轮询，按服务器权重顺序循环分发请求，支持运行时动态调整权重，权重高的服务器优先分发更多请求|动态可调，负载分配均匀，支持无限服务器节点，默认算法|短连接场景表现最优，长连接场景负载均衡效果有限|通用 HTTP 短连接业务，服务器配置相近的通用场景|
|静态轮询算法|`balance static-rr`|静态加权轮询，权重在配置加载时固定，运行时不可调整，无调度开销|零调度开销，性能略高于 roundrobin|不支持运行时权重调整，服务器节点数量有限制|高并发静态 HTTP 服务，服务器配置固定无动态调整需求的场景|
|最小连接数算法|`balance leastconn`|动态加权最小连接数，将请求分发到当前活跃连接数最少的服务器，权重参与计算|适配长连接场景，自动平衡服务器负载，避免慢服务器过载|短连接场景下连接数变化快，调度效果不如轮询|TCP 长连接服务，如 MySQL、Redis、SSH，慢接口 HTTP 服务|
|源地址哈希算法|`balance source`|基于客户端源 IP 地址进行哈希计算，将同一源 IP 的请求始终分发到同一后端服务器，哈希环随服务器节点上下线动态调整|实现无 Cookie 的会话保持，无需解析应用层数据，TCP/HTTP 模式均支持|服务器节点变化会导致哈希环重分布，会话失效；客户端 IP 集中时会导致负载不均|无 Cookie 的 TCP 服务会话保持，不支持 Cookie 的客户端场景|
|URI 哈希算法|`balance uri`|基于 HTTP 请求的 URI 进行哈希计算，将同一 URI 的请求始终分发到同一后端服务器|提升缓存服务器的命中率，适配静态资源服务|仅 HTTP 模式生效，服务器节点变化会导致哈希环重分布|缓存服务器、CDN 回源、静态资源服务场景|
|请求头哈希算法|`balance hdr(<请求头名称>)`|基于指定 HTTP 请求头的值进行哈希计算，如`hdr(Cookie)`、`hdr(Host)`|实现精细化的会话保持与内容路由，支持自定义维度|仅 HTTP 模式生效，请求头不存在时回退到轮询算法|基于域名、用户 ID、Cookie 的精细化流量调度场景|
|随机算法|`balance random`|基于权重随机分发请求，支持动态权重调整|大规模服务器集群场景下，负载分配均匀度优于轮询，避免轮询算法的 "惊群" 效应|小规模集群场景下，负载均匀度不如轮询|超大规模服务器集群（超过 100 个节点）、容器化动态扩缩容场景|

#### `server` 指令核心扩展参数

`server`指令是后端配置的核心，其扩展参数直接决定服务器的调度、健康检查、连接管理规则，生产环境必备核心参数如下：

- `weight <数值>`：设置服务器的权重，取值范围 0-256，默认 1，权重越高分发的请求越多，0 表示不接收新请求
- `check`：启用该服务器的健康检查，未配置则仅在 TCP 连接失败时标记为故障
- `inter <时间>`：设置该服务器的健康检查间隔，覆盖 defaults 段的配置
- `rise <次数>`：设置故障服务器恢复上线的连续成功次数，覆盖 defaults 段的配置
- `fall <次数>`：设置健康服务器故障下线的连续失败次数，覆盖 defaults 段的配置
- `maxconn <数值>`：设置该服务器的最大并发连接数，超过的请求进入队列等待
- `maxqueue <数值>`：设置该服务器的最大队列长度，超过队列长度的请求返回 503 错误
- `backup`：标记该服务器为备份服务器，仅当所有非备份服务器均故障下线时，才会接收请求
- `disabled`：标记该服务器为禁用状态，不接收任何请求，用于服务器下线维护
- `send-proxy`：启用 PROXY 协议发送，将客户端真实 IP 传递给后端服务器，需后端服务支持 PROXY 协议
- `ssl`：启用与后端服务器的 TLS 加密连接，用于后端 HTTPS 服务的转发
- `verify <none/required>`：设置后端 TLS 证书校验模式，`required`为强制校验证书有效性，生产环境推荐配置

### 2.3.5 `listen` 监听配置段

`listen`段是`frontend`与`backend`的合一简化段，同时包含流量入口监听与后端服务器池配置，无需配置`default_backend`与`use_backend`，配置更简洁，无功能差异。

#### 适用场景

1. TCP 服务的负载均衡，如 MySQL、Redis、SSH 等，无需复杂 7 层路由规则
2. 简单 HTTP 服务，单入口对应单后端池，无多路由分发需求
3. 监控统计页面、运维接口等单一功能服务

#### 语法规范

`listen`段的指令体系完全兼容`frontend`与`backend`段，核心语法格式：

plaintext

```
listen <段名称> <监听地址>:<端口>
    # 基础配置
    mode <tcp/http>
    # 负载均衡配置
    balance <算法>
    # 服务器定义
    server <名称> <地址>:<端口> <参数>
    # 其他兼容frontend/backend的所有指令
```

## 2.4 配置生效与热重载机制

HAProxy 支持零停机配置热重载，核心原理基于第一章的 Master-Worker 架构，完整机制与操作规范如下：

### 2.4.1 配置生效规则

HAProxy 配置分为三类，不同类型的配置生效方式不同，无隐式生效规则：

1. **热重载生效配置**：所有`defaults`/`frontend`/`backend`/`listen`段的配置，包括监听端口、路由规则、负载均衡算法、服务器节点、超时配置等，均可通过热重载生效，无需重启进程，不中断业务流量
2. **重启生效配置**：`global`段的核心底层配置，包括`nbproc`/`nbthread`/`cpu-map`/`chroot`/`user`/`group`/`ulimit-n`/`tune.*`系列性能调优参数，必须重启 Master 进程才能生效，热重载无法更新
3. **运行时动态生效配置**：部分配置可通过`stats socket`CLI 接口在运行时动态调整，无需热重载，包括服务器权重、上下线状态、健康检查参数、ACL 规则等，适用于临时调整、灰度发布、故障应急场景

### 2.4.2 热重载核心操作命令

表格

|命令|核心作用|适用场景|
|---|---|---|
|`haproxy -c -f <配置文件路径>`|配置文件语法与合法性校验，校验通过返回 0，失败返回非 0|配置修改后必选执行，避免热重载失败|
|`haproxy -f <配置文件路径> -sf $(cat <pidfile路径>)`|优雅热重载，启动新的 Master-Worker 进程，旧进程完成现有连接处理后优雅退出|生产环境标准配置更新命令，零停机|
|`haproxy -f <配置文件路径> -st $(cat <pidfile路径>)`|强制重启，立即终止旧进程，中断现有连接|仅紧急场景使用，生产环境禁止常规使用|

### 2.4.3 热重载技术原理

1. Master 进程接收到热重载信号后，fork 新的 Master 子进程，传递监听套接字文件描述符
2. 新 Master 进程解析并校验配置文件，校验通过后启动新的 Worker 进程，接管监听套接字，开始接收新的客户端请求
3. 旧 Master 进程向旧 Worker 进程发送优雅退出信号，旧 Worker 进程停止接收新请求，完成现有连接的处理后，正常关闭连接并退出
4. 所有旧 Worker 进程退出后，旧 Master 进程终止，新 Master 进程成为实例唯一主进程，热重载完成

## 2.5 配置校验与调试规范

### 2.5.1 基础语法校验

配置修改后必须先执行语法校验，命令如下：

bash

运行

```
# 基础语法校验
haproxy -c -f /etc/haproxy/haproxy.cfg
# 详细校验，输出配置解析细节与警告信息
haproxy -c -V -f /etc/haproxy/haproxy.cfg
# 校验指定分段的配置有效性
haproxy -c -f /etc/haproxy/haproxy.cfg -d
```

- 校验失败时，HAProxy 会精准输出错误的行号、错误类型与原因，可直接定位问题
- 警告信息不影响配置生效，但生产环境建议处理所有警告，避免潜在风险

### 2.5.2 调试模式

HAProxy 提供前台调试模式，可实时输出流量处理、连接状态、规则匹配的详细日志，用于配置调试与问题排查，命令如下：

bash

运行

```
# 前台调试模式，输出详细调试日志，Ctrl+C终止
haproxy -f /etc/haproxy/haproxy.cfg -d
```

- 调试模式下，HAProxy 不会以守护进程运行，所有日志实时输出到终端
- 仅用于测试环境调试，生产环境禁止启用，会导致性能严重下降

## 2.6 最小可用配置模板

本节提供两个生产级最小可用配置模板，覆盖 TCP 与 HTTP 两大核心场景，完全符合本章的配置规范，可直接用于学习验证与生产环境基础部署。

### 2.6.1 HTTP 模式最小可用配置

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
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384

# 默认配置段
defaults
    mode http
    log global
    option httplog
    option forwardfor
    option http-server-close
    option redispatch
    retries 3
    timeout connect 5s
    timeout client 30s
    timeout server 30s
    timeout http-request 10s
    timeout http-keep-alive 5s
    timeout queue 30s
    timeout check 2s
    balance roundrobin
    inter 2s
    rise 2
    fall 3

# 前端HTTP入口
frontend http_front
    bind 0.0.0.0:80
    default_backend http_back

# 后端HTTP服务池
backend http_back
    server web01 192.168.1.101:80 check weight 1
    server web02 192.168.1.102:80 check weight 1
```

### 2.6.2 TCP 模式最小可用配置（MySQL 负载均衡）

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

# 默认配置段
defaults
    mode tcp
    log global
    option tcplog
    retries 3
    timeout connect 5s
    timeout client 1800s
    timeout server 1800s
    timeout check 2s
    balance leastconn
    inter 2s
    rise 2
    fall 3

# 监听段（TCP合一配置）
listen mysql_proxy
    bind 0.0.0.0:3306
    server mysql01 192.168.1.201:3306 check weight 1 maxconn 1000
    server mysql02 192.168.1.202:3306 check weight 1 maxconn 1000 backup
```