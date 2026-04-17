(acl [名称] [匹配标准] [标志] [操作符] [值])

# 第五章 HAProxy SSL/TLS 全栈能力与安全防护体系

本章为 HAProxy 生产环境落地的核心安全章节，完整定义其 SSL/TLS 全链路加密能力、证书生命周期管理、协议安全规范与全栈攻击防护体系，所有内容均基于 2.8.x LTS 版本官方标准与 TLS 1.3 RFC 规范，严格遵循网络安全学术设计原则，无冗余表述与非标准化配置，完整承接前四章的技术体系。

## 5.1 SSL/TLS 核心架构与工作原理

### 5.1.1 核心定义与角色定位

HAProxy 作为流量入口的 SSL/TLS 终结点，是**客户端与上游服务之间的 TLS 信任锚点与加密流量管控核心**，其核心职责分为三大类：

1. **TLS 卸载（终结）**：在 HAProxy 侧完成客户端 TLS 握手、加解密运算，将解密后的明文 HTTP 流量转发给上游服务，彻底释放上游服务的 TLS 运算压力，统一管控证书与加密策略
2. **端到端 TLS 加密**：同时完成客户端侧 TLS 终结与服务端侧 TLS 发起，实现客户端到 HAProxy、HAProxy 到上游服务的全链路 TLS 加密，无明文传输环节，满足高安全合规要求
3. **TLS 透传与路由**：四层模式下不解密 TLS 流量，仅通过 TLS 握手报文的 SNI（Server Name Indication）扩展字段实现流量路由，全程不触碰加密内容，适配证书私有化、合规性强隔离的业务场景

### 5.1.2 三大 SSL/TLS 工作模式与技术边界

HAProxy 的 SSL/TLS 能力严格区分三大工作模式，各模式的能力边界、性能开销、适用场景有明确的学术级区分，无模糊交叉：

表格

|工作模式|OSI 工作层级|核心处理逻辑|证书持有要求|性能开销|核心能力边界|最佳适用场景|
|---|---|---|---|---|---|---|
|七层 TLS 卸载模式|应用层 L7|完整终结客户端 TLS 连接，解密 HTTP 流量，执行七层规则后转发明文 / 加密流量到上游|必须持有服务端证书与私钥|中高，主要开销为 TLS 握手与加解密运算|支持全量七层能力：内容路由、请求改写、Cookie 会话保持、应用层防护|通用 Web 业务、API 网关、需精细化七层管控的场景，生产环境主流模式|
|端到端 TLS 加密模式|应用层 L7|客户端侧终结 TLS，服务端侧发起新的 TLS 连接，全链路无明文传输|持有服务端证书，同时信任上游服务的 CA 证书|高，双向 TLS 加解密运算|保留全量七层能力，同时满足全链路加密合规要求|金融、政务等高安全合规场景，跨公网服务调用场景|
|四层 TLS 透传模式|传输层 L4|不解密 TLS 流量，仅解析 TLS 握手报文的 SNI 字段实现路由，双向透传加密流量|无需持有任何证书与私钥|极低，仅解析 TLS 握手头部，无加解密开销|无七层能力，仅支持四层流量管控，无法修改加密内容|多租户 HTTPS 服务隔离、证书私有化部署、国密算法适配场景|

### 5.1.3 HAProxy TLS 握手全流程处理机制

HAProxy 严格遵循 TLS 协议规范实现握手流程，同时针对高并发场景做了深度优化，完整握手处理流程如下（以 TLS 1.3 为例）：

1. **TCP 连接建立**：客户端与 HAProxy 完成 TCP 三次握手，进入 TLS 握手阶段
2. **Client Hello 解析**：HAProxy 接收客户端 Client Hello 报文，提取 TLS 版本、加密套件、SNI、ALPN、会话 Ticket 等扩展字段，执行以下校验：
    
    - TLS 版本合规性校验，拒绝低于安全基线的协议版本
    - 基于 SNI 匹配对应的服务端证书，无 SNI 则使用默认证书
    - 会话复用匹配，优先恢复已有会话，跳过完整握手
    
3. **Server Hello 响应**：HAProxy 向客户端返回 Server Hello 报文，协商最终的 TLS 版本、加密套件、密钥交换参数，同时发送服务器证书、证书链与 OCSP 装订数据
4. **握手完成与密钥派生**：客户端与 HAProxy 完成密钥交换，派生会话密钥，双向发送 Finished 报文，握手完成
5. **应用层数据传输**：HAProxy 使用协商的会话密钥完成应用层数据的加解密，执行七层流量管控规则，转发流量到上游服务

**核心优化点**：HAProxy 将 TLS 握手的非对称加密运算、证书校验等重负载操作，均匀分发到多线程 / 多进程中，无锁竞争，实现多核 CPU 的线性性能扩展。

### 5.1.4 TLS 协议版本兼容性与安全基线

HAProxy 完整支持 SSL 3.0 至 TLS 1.3 全系列协议版本，基于当前网络安全学术规范与合规要求，生产环境必须遵循以下安全基线：

1. **强制禁用版本**：SSL 3.0、TLS 1.0、TLS 1.1，上述版本存在不可修复的安全漏洞（如 POODLE、BEAST、CRIME），不符合等保 2.0、PCI-DSS 等合规要求
2. **推荐启用版本**：TLS 1.2、TLS 1.3，其中 TLS 1.3 为优先推荐版本，握手延迟降低 50%，安全性显著提升，兼容 98% 以上的现代客户端
3. **兼容性兜底**：面向老旧客户端（如 IE11、旧版 Android）的业务，可保留 TLS 1.2 作为兜底，禁止启用更低版本

## 5.2 SSL/TLS 基础配置规范与核心参数

### 5.2.1 证书格式规范与准备要求

HAProxy 仅支持**PEM 格式**的证书与私钥文件，对证书格式、权限有严格的约束，不符合要求会直接导致配置加载失败，生产环境必须遵循以下规范：

1. **证书文件结构**：单个 PEM 文件需按顺序包含以下内容，不可打乱顺序：
    
    1. 服务端域名证书（叶子证书）
    2. 中间 CA 证书（证书链），按层级从上到下排列
    3. 服务端私钥（可选，可单独存放于独立文件）
    
2. **私钥安全规范**：
    
    - 私钥必须为 RSA 2048 位及以上、或 ECDSA P-256 及以上密钥长度，禁止使用 1024 位及以下弱密钥
    - 私钥文件权限必须为`600`，所属用户与组必须与 HAProxy 运行用户一致，禁止全局可读
    - 生产环境禁止使用未加密的私钥，推荐使用加密私钥，通过`ssl-password-file`指定密码文件
    
3. **证书校验要求**：
    
    - 证书必须包含完整的证书链，避免客户端出现证书不可信错误
    - 证书 SAN 字段必须覆盖所有绑定的域名，通配符证书仅支持一级子域名匹配
    - 证书密钥用法必须包含`TLS Web Server Authentication`，密钥交换算法与加密套件兼容
    

### 5.2.2 全局 SSL/TLS 默认配置

全局 SSL 配置在`global`段中定义，为所有前端监听端口与后端服务连接提供统一的安全基线，避免重复配置，生产环境必须显式定义，禁止使用默认值。

#### 生产级全局标准配置

plaintext

```
global
    # 前端监听端口默认TLS选项：禁用不安全协议版本
    ssl-default-bind-options no-sslv3 no-tlsv10 no-tlsv11 no-tls-tickets
    # 前端TLS 1.2加密套件：仅启用强加密套件，按优先级排序
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305
    # 前端TLS 1.3加密套件：独立配置，仅支持AEAD强加密套件
    ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
    # 后端服务连接默认TLS选项
    ssl-default-server-options no-sslv3 no-tlsv10 no-tlsv11
    # 后端TLS加密套件，与前端基线一致
    ssl-default-server-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384
    ssl-default-server-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
    # DH参数位数，2048位为安全基线，禁止使用1024位
    tune.ssl.default-dh-param 2048
    # TLS记录层缓冲区优化，平衡性能与内存占用
    tune.ssl.maxrecord 16384
```

#### 核心全局参数约束

表格

|参数|核心作用|强制约束|
|---|---|---|
|`ssl-default-bind-options`|定义所有前端`bind`指令的默认 TLS 行为|必须禁用`sslv3`/`tlsv10`/`tlsv11`，生产环境强制要求|
|`ssl-default-bind-ciphers`|定义 TLS 1.2 及以下版本的默认加密套件|仅启用 AEAD 认证加密套件，禁用 CBC、RC4 等弱加密套件|
|`ssl-default-bind-ciphersuites`|定义 TLS 1.3 版本的专属加密套件|仅支持 TLS 1.3 标准定义的 AEAD 套件，不可与 TLS 1.2 套件混用|
|`tune.ssl.default-dh-param`|定义 DHE 密钥交换的 DH 参数位数|最小值 2048，高安全场景使用 4096，禁止 1024 位及以下|
|`tune.ssl.maxrecord`|定义 TLS 记录层的最大分片大小|推荐 16384 字节，过大导致延迟增加，过小导致性能下降|

### 5.2.3 客户端侧 TLS 终结（卸载）配置

客户端侧 TLS 卸载在`frontend`段的`bind`指令中配置，是 HTTPS 服务的核心基础配置，核心作用是监听 443 端口，启用 TLS 加密，绑定服务端证书。

#### 最小可用标准配置

plaintext

```
frontend https_front
    # 启用TLS卸载，绑定证书，配置ALPN协议协商
    bind 0.0.0.0:443 ssl \
        crt /etc/haproxy/certs/www.example.com.pem \
        alpn h2,http/1.1 \
        strict-sni \
        no-tls-tickets
    mode http
    # 继承全局默认配置，可单独覆盖
    default_backend web_back
```

#### `bind`指令 SSL 核心扩展参数

表格

|参数|核心作用|生产环境建议|
|---|---|---|
|`ssl`|启用该端口的 TLS 加密能力|必选参数，无此参数则为明文 HTTP|
|`crt <证书路径>`|指定服务端证书文件路径，可配置多个实现 SNI 多域名支持|必选参数，支持目录批量加载：`crt /etc/haproxy/certs/`|
|`alpn <协议列表>`|指定 TLS ALPN 扩展的应用层协议协商列表，用于 HTTP/2/3 协商|推荐配置`h2,http/1.1`，优先协商 HTTP/2|
|`strict-sni`|强制 SNI 校验，客户端未携带 SNI 时直接拒绝 TLS 握手|多域名场景推荐配置，避免使用默认证书泄露域名信息|
|`no-tls-tickets`|禁用 TLS 会话 Ticket，仅使用 Session ID 实现会话复用|高安全场景推荐配置，避免会话密钥泄露风险|
|`ca-file <CA证书路径>`|指定客户端证书的 CA 根证书，用于 mTLS 双向认证|mTLS 场景必选参数|
|`verify <none/optional/required>`|定义客户端证书校验模式|mTLS 场景必须设置为`required`，强制校验客户端证书|
|`ssl-max-ver <版本>`|该端口支持的最高 TLS 版本|推荐`TLSv1.3`|
|`ssl-min-ver <版本>`|该端口支持的最低 TLS 版本|推荐`TLSv1.2`，兼容老旧客户端可设置为`TLSv1.1`|

### 5.2.4 服务端侧 TLS 端到端加密配置

服务端侧 TLS 配置在`backend`段的`server`指令中配置，核心作用是 HAProxy 与上游服务之间建立 TLS 加密连接，实现端到端全链路加密。

#### 标准配置示例

plaintext

```
backend web_back
    mode http
    balance roundrobin
    option httpchk GET /health HTTP/1.1
    http-check send hdr Host www.example.com
    http-check expect status 200
    # 端到端TLS配置：ssl启用加密，verify required强制校验服务端证书，ca-file指定信任的CA根证书
    server web01 192.168.1.101:443 check ssl verify required ca-file /etc/haproxy/certs/upstream_ca.pem
    server web02 192.168.1.102:443 check ssl verify required ca-file /etc/haproxy/certs/upstream_ca.pem
```

#### `server`指令 SSL 核心扩展参数

表格

|参数|核心作用|安全约束|
|---|---|---|
|`ssl`|启用与上游服务的 TLS 加密连接|必选参数，无此参数则为明文 TCP 连接|
|`verify required`|强制校验上游服务的 TLS 证书有效性，证书无效则拒绝连接|生产环境必选配置，禁止使用`verify none`跳过校验，避免中间人攻击|
|`ca-file <CA路径>`|指定信任的上游服务 CA 根证书，用于证书校验|`verify required`场景必选参数，必须包含完整的 CA 证书链|
|`crt <客户端证书路径>`|指定 HAProxy 向上游服务出示的客户端证书，用于上游服务的 mTLS 认证|上游服务要求双向认证时必选|
|`sni <域名>`|指定向上游服务 TLS 握手时携带的 SNI 域名|上游服务为虚拟主机场景必选，推荐使用`%[req.hdr(Host)]`动态传递|
|`ssl-max-ver`/`ssl-min-ver`|定义与上游服务通信的 TLS 版本范围|与前端安全基线保持一致，最低 TLS 1.2|

### 5.2.5 HTTP 到 HTTPS 强制跳转配置

生产环境必须将 80 端口的明文 HTTP 请求强制跳转到 HTTPS，确保所有流量均为加密传输，标准配置如下：

plaintext

```
frontend http_front
    bind 0.0.0.0:80
    mode http
    # 定义ACL，排除内网健康检查与ACME证书验证请求
    acl internal_check src 10.0.0.0/8 192.168.0.0/16
    acl acme_challenge path_beg /.well-known/acme-challenge/
    # ACME验证请求转发到证书签发工具
    use_backend acme_back if acme_challenge
    # 内网健康检查直接放行，其余请求301永久跳转到HTTPS
    http-request redirect scheme https code 301 if !internal_check !acme_challenge
    default_backend web_back
```

## 5.3 高级 SSL/TLS 特性与生产级实现

### 5.3.1 SNI 多域名证书体系与路由配置

SNI（Server Name Indication）是 TLS 协议的扩展字段，允许客户端在 TLS 握手时携带目标域名，HAProxy 基于 SNI 实现**单 IP 单端口绑定多域名多证书**，是多站点 HTTPS 服务的核心解决方案。

#### 1. 批量证书加载与自动 SNI 匹配

HAProxy 支持证书目录批量加载，自动解析证书中的域名与 SAN 字段，实现 SNI 自动匹配，无需为每个域名单独配置，生产环境推荐用法：

plaintext

```
frontend https_front
    bind 0.0.0.0:443 ssl \
        # 批量加载证书目录下的所有PEM证书，自动匹配SNI
        crt /etc/haproxy/certs/ \
        # 默认证书，客户端无SNI时使用
        crt /etc/haproxy/certs/default.example.com.pem \
        alpn h2,http/1.1 \
        ssl-min-ver TLSv1.2
    mode http
    # 基于SNI域名的路由规则
    acl site_www ssl_fc_sni -i www.example.com example.com
    acl site_api ssl_fc_sni -i api.example.com
    acl site_mobile ssl_fc_sni -i m.example.com
    use_backend www_back if site_www
    use_backend api_back if site_api
    use_backend mobile_back if site_mobile
    default_backend www_back
```

#### 2. 核心 SNI 匹配规则

1. **匹配优先级**：客户端携带 SNI 时，优先匹配证书 SAN 字段完全一致的证书；无匹配时使用默认证书
2. **通配符匹配**：支持通配符证书（如`*.example.com`），仅匹配一级子域名，不可跨级匹配
3. **ACL 匹配因子**：`ssl_fc_sni`用于提取客户端 TLS 握手的 SNI 域名，可结合 ACL 实现精细化路由与访问控制
4. **严格校验**：`strict-sni`参数启用后，无 SNI 或 SNI 无匹配证书时，直接拒绝 TLS 握手，避免默认证书泄露

### 5.3.2 双向 TLS（mTLS）客户端证书认证

双向 TLS（mutual TLS，mTLS）是高安全场景的核心认证方式，不仅客户端校验服务端证书，服务端也强制校验客户端证书，实现双向身份认证，彻底杜绝未授权访问，适用于 API 网关、内部服务调用、金融级业务场景。

#### 1. mTLS 核心原理

mTLS 在标准 TLS 握手基础上，增加了服务端对客户端的证书校验流程：

1. 客户端发送 Client Hello，服务端返回 Server Hello 与服务端证书
2. 服务端向客户端发送 Certificate Request 报文，要求客户端出示证书
3. 客户端向服务端发送客户端证书与证书验证信息
4. 服务端校验客户端证书的有效性、CA 签发信任链、证书吊销状态
5. 校验通过后，完成 TLS 握手，建立加密连接

#### 2. 生产级 mTLS 标准配置

plaintext

```
frontend mtls_api_front
    bind 0.0.0.0:443 ssl \
        crt /etc/haproxy/certs/api.example.com.pem \
        # 客户端CA根证书，用于校验客户端证书
        ca-file /etc/haproxy/certs/client_ca.pem \
        # 强制校验客户端证书，校验失败直接拒绝握手
        verify required \
        # 启用CRL证书吊销列表校验
        crl-file /etc/haproxy/certs/client_crl.pem \
        alpn h2,http/1.1 \
        ssl-min-ver TLSv1.2
    mode http
    # 基于客户端证书信息的ACL访问控制
    acl valid_client_cert ssl_c_verify_result eq 0
    acl admin_client ssl_c_s_dn(CN) -i admin-api
    acl service_client ssl_c_s_dn(CN) -i service-api
    # 证书校验失败直接拒绝
    http-request deny status 403 if !valid_client_cert
    # 基于客户端证书CN字段路由
    use_backend admin_api_back if admin_client
    use_backend service_api_back if service_client
    default_backend service_api_back
```

#### 3. 客户端证书核心 ACL 匹配因子

表格

|匹配因子|核心作用|适用场景|
|---|---|---|
|`ssl_c_verify_result`|客户端证书校验结果，0 为校验成功，非 0 为失败|证书有效性校验，拒绝无效证书请求|
|`ssl_c_s_dn(<字段>)`|提取客户端证书使用者 DN 字段，如 CN、O、OU、C|基于证书主体信息的身份认证与访问控制|
|`ssl_c_i_dn(<字段>)`|提取客户端证书签发者 DN 字段，如 CN、O|区分不同 CA 签发的证书，实现多 CA 分级管控|
|`ssl_c_serial`|提取客户端证书的序列号|实现单证书粒度的黑白名单管控|
|`ssl_c_notbefore`/`ssl_c_notafter`|提取客户端证书的有效期起止时间|拦截过期证书与未生效证书|

### 5.3.3 TLS 会话复用机制与性能优化

TLS 握手的非对称加密运算为 CPU 密集型操作，会话复用机制可跳过完整握手流程，直接复用已有会话密钥，将 TLS 握手耗时从 2-RTT 降低到 0-RTT，CPU 开销降低 90% 以上，是高并发 HTTPS 场景的核心性能优化手段。

HAProxy 支持三种会话复用机制，技术特性与适用场景严格区分：

表格

|复用机制|技术原理|会话存储位置|集群同步能力|安全性|性能|最佳适用场景|
|---|---|---|---|---|---|---|
|Session ID|客户端与服务端通过 32 位 Session ID 标识会话，服务端存储会话密钥|HAProxy 本地内存|不支持|高，会话密钥仅存储在服务端|高|单节点部署场景，高安全要求业务|
|Session Ticket|服务端用 Ticket 密钥加密会话密钥，生成 Ticket 发送给客户端，客户端后续握手携带 Ticket|客户端本地|支持，集群内共享 Ticket 密钥|中，Ticket 密钥泄露会导致会话密钥泄露|极高|多节点集群部署，高并发大流量场景|
|TLS 1.3 0-RTT|TLS 1.3 专属特性，基于 PSK 预共享密钥，客户端在第一个报文中即可携带应用数据|客户端本地|支持|中，存在重放攻击风险|最高|静态资源、GET 等幂等请求场景|

#### 生产级会话复用配置

1. **Session ID 配置（单节点高安全场景）**
    
    plaintext
    
    ```
    global
        # 会话ID缓存大小，最大存储100万条会话，30分钟过期
        tune.ssl.cachesize 1000000
        tune.ssl.lifetime 1800
    frontend https_front
        bind 0.0.0.0:443 ssl \
            crt /etc/haproxy/certs/www.example.com.pem \
            alpn h2,http/1.1 \
            no-tls-tickets
    ```
    
2. **Session Ticket 配置（集群高并发场景）**
    
    plaintext
    
    ```
    global
        # 全局共享Ticket密钥文件，集群所有节点使用相同密钥
        ssl-default-bind-options ssl-prefer-server-ciphers no-tls-tickets
        ssl-ticket-key /etc/haproxy/certs/ssl-ticket.key
    frontend https_front
        bind 0.0.0.0:443 ssl \
            crt /etc/haproxy/certs/www.example.com.pem \
            alpn h2,http/1.1 \
            tls-tickets
    ```
    
3. **TLS 1.3 0-RTT 配置（幂等请求场景）**
    
    plaintext
    
    ```
    frontend https_front
        bind 0.0.0.0:443 ssl \
            crt /etc/haproxy/certs/www.example.com.pem \
            alpn h2,http/1.1 \
            ssl-min-ver TLSv1.3 \
            # 启用0-RTT
            early-data
        mode http
        # 仅允许GET/HEAD等幂等请求使用0-RTT，防止重放攻击
        acl idempotent_method method GET HEAD OPTIONS
        http-request deny early-data if !idempotent_method
    ```
    

### 5.3.4 OCSP Stapling 证书吊销状态装订

OCSP（Online Certificate Status Protocol）是证书吊销状态查询协议，传统 OCSP 查询由客户端向 CA 服务器发起，存在隐私泄露、查询延迟、查询失败导致证书不可信等问题。OCSP Stapling（OCSP 装订）由 HAProxy 代替客户端向 CA 服务器查询证书吊销状态，将 OCSP 响应装订在 TLS 握手报文中发送给客户端，彻底解决上述问题，同时满足合规要求。

#### 生产级 OCSP Stapling 配置

plaintext

```
frontend https_front
    bind 0.0.0.0:443 ssl \
        crt /etc/haproxy/certs/www.example.com.pem \
        alpn h2,http/1.1 \
        ssl-min-ver TLSv1.2 \
        # 启用OCSP Stapling
        ocsp-update on \
        # OCSP响应强制校验，校验失败不装订
        ocsp-strict on \
        # 指定CA根证书，用于OCSP响应签名校验
        ca-file /etc/haproxy/certs/root_ca.pem
```

#### 核心执行规则

1. HAProxy 启动时自动向证书的 OCSP 服务器发起查询，获取 OCSP 响应
2. 按 OCSP 响应的有效期自动更新，默认提前 1 小时刷新，避免过期
3. TLS 握手时，将 OCSP 响应随证书一起发送给客户端，客户端无需额外发起 OCSP 查询
4. `ocsp-strict on`启用后，OCSP 查询失败或响应无效时，禁止使用该证书，避免吊销证书被滥用

### 5.3.5 HTTP/2 & HTTP/3 over TLS 配置

HAProxy 完整支持 HTTP/2 与 HTTP/3 协议，均基于 TLS 加密传输，可显著提升 Web 业务的访问性能，生产环境配置规范如下：

#### 1. HTTP/2 over TLS 配置

HTTP/2 基于 TLS 1.2 + 协议，通过 ALPN 扩展协商，多路复用特性可显著降低高并发请求的延迟，生产环境标准配置：

plaintext

```
frontend https_front
    bind 0.0.0.0:443 ssl \
        crt /etc/haproxy/certs/www.example.com.pem \
        # ALPN协商，优先HTTP/2，降级HTTP/1.1
        alpn h2,http/1.1 \
        ssl-min-ver TLSv1.2
    mode http
    # HTTP/2专用超时优化
    timeout http-keep-alive 10s
    timeout client 30s
    default_backend web_back
```

#### 2. HTTP/3 over QUIC 配置

HTTP/3 基于 QUIC 协议，运行在 UDP 之上，彻底解决 TCP 队头阻塞问题，握手延迟更低，2.8 + 版本原生支持，生产环境配置规范：

plaintext

```
global
    # 启用QUIC协议支持
    tune.quic.frontend.max-conn 100000
    tune.quic.frontend.max-streams-bidi 100
frontend https_front
    # TCP+HTTP/2监听
    bind 0.0.0.0:443 ssl \
        crt /etc/haproxy/certs/www.example.com.pem \
        alpn h3,h2,http/1.1 \
        ssl-min-ver TLSv1.3
    # UDP+HTTP/3监听，配置与TCP端口一致
    bind quic4@0.0.0.0:443 ssl \
        crt /etc/haproxy/certs/www.example.com.pem \
        alpn h3 \
        ssl-min-ver TLSv1.3
    mode http
    # 向客户端发送Alt-Svc响应头，告知HTTP/3服务地址
    http-response set-header Alt-Svc 'h3=":443"; ma=86400'
    default_backend web_back
```

### 5.3.6 动态证书管理与无中断更新

生产环境证书更新是高频运维操作，HAProxy 2.2 + 版本支持动态证书管理，无需重启或热重载进程，即可完成证书的新增、更新、删除，实现零业务中断的证书生命周期管理。

#### 1. 动态证书更新核心操作

通过 HAProxy 的 stats socket CLI 接口实现动态证书管理，核心命令如下：

bash

运行

```
# 1. 查看当前加载的证书列表
echo "show ssl cert" | socat /var/run/haproxy.sock stdio

# 2. 更新已有证书，零中断生效
echo "set ssl cert /etc/haproxy/certs/www.example.com.pem <<\n$(cat /etc/haproxy/certs/new_www.example.com.pem)\n" | socat /var/run/haproxy.sock stdio

# 3. 提交证书更新，正式生效
echo "commit ssl cert /etc/haproxy/certs/www.example.com.pem" | socat /var/run/haproxy.sock stdio

# 4. 新增证书
echo "add ssl cert /etc/haproxy/certs/new.example.com.pem <<\n$(cat /etc/haproxy/certs/new.example.com.pem)\n" | socat /var/run/haproxy.sock stdio
echo "commit ssl cert /etc/haproxy/certs/new.example.com.pem" | socat /var/run/haproxy.sock stdio
```

#### 2. 证书自动续期适配

可配合 Certbot、acme.sh 等 ACME 客户端实现证书自动续期，续期完成后通过上述 CLI 命令动态更新证书，无需重启 HAProxy，彻底解决证书过期问题。

### 5.3.7 四层 TLS 透传与 SNI 路由配置

四层 TLS 透传模式下，HAProxy 不解密 TLS 流量，仅解析 Client Hello 报文中的 SNI 字段实现路由，全程透传加密流量，无需持有证书，适用于多租户隔离、证书私有化、国密算法适配等场景。

#### 生产级四层 TLS 透传配置

plaintext

```
frontend tls_passthrough_front
    bind 0.0.0.0:443
    mode tcp
    # 等待客户端SSL Hello报文接收完成，提取SNI字段
    tcp-request inspect-delay 5s
    tcp-request content accept if { req_ssl_hello_type 1 }
    # 基于SNI域名的ACL路由
    acl site_www req_ssl_sni -i www.example.com
    acl site_api req_ssl_sni -i api.example.com
    # 基于SNI路由到对应后端，透传TLS流量
    use_backend www_tls_back if site_www
    use_backend api_tls_back if site_api
    default_backend default_tls_back

# 后端服务池，透传TLS流量，不解密
backend www_tls_back
    mode tcp
    balance leastconn
    server web01 192.168.1.101:443 check
    server web02 192.168.1.102:443 check

backend api_tls_back
    mode tcp
    balance leastconn
    server api01 192.168.1.151:443 check
    server api02 192.168.1.152:443 check
```

#### 核心约束与注意事项

1. **inspect-delay**：必须配置`tcp-request inspect-delay`，给 HAProxy 预留足够时间接收 SSL Hello 报文，否则无法提取 SNI 字段
2. **req_ssl_hello_type 1**：仅当客户端发送 SSL Hello 报文（类型 1）时才接受连接，避免非 TLS 流量进入
3. **能力边界**：四层透传模式下，无法执行任何七层规则，包括 HTTP 请求改写、内容路由、应用层防护，仅支持四层流量管控
4. **健康检查**：后端服务为 HTTPS 端口，需配置 SSL 健康检查，`check ssl`参数启用 TLS 健康检查

## 5.4 全栈安全防护体系

HAProxy 作为业务流量的唯一入口，是网络安全防护的第一道防线，提供从网络层到应用层的全栈防护能力，可抵御绝大多数常见的网络攻击与恶意请求，生产环境必须构建完整的防护体系。

### 5.4.1 网络层 DDoS 攻击防护

网络层 DDoS 攻击的核心目标是耗尽服务器的带宽、连接数、CPU 资源，HAProxy 通过内核优化、连接限制、流量整形实现基础防护，核心配置如下：

#### 生产级 DDoS 防护配置

plaintext

```
global
    # 限制单个进程的最大并发连接数，避免资源耗尽
    maxconn 100000
    # 启用TCP SYN Cookie，抵御SYN洪水攻击
    tune.tcp.syn_cookies on
    # 限制单个源IP的最大并发连接数
frontend http_front
    bind 0.0.0.0:80
    bind 0.0.0.0:443 ssl crt /etc/haproxy/certs/www.example.com.pem
    mode http
    # 定义粘性表，跟踪源IP的连接数、新建连接速率
    stick-table type ip size 200k expire 10m store conn_cur,conn_rate(10s),http_req_rate(10s)
    http-request track-sc0 src
    # 定义攻击检测ACL
    acl syn_flood sc0_conn_rate gt 100
    acl too_many_connections sc0_conn_cur gt 50
    acl cc_attack sc0_http_req_rate gt 200
    # 攻击防护动作
    http-request silent-drop if syn_flood
    http-request deny status 429 if too_many_connections
    http-request tarpit if cc_attack
    # TCP层连接管控
    tcp-request content reject if { src_conn_cur gt 100 }
```

#### 核心防护能力

1. **SYN 洪水防护**：通过`tune.tcp.syn_cookies on`启用 SYN Cookie，无需为半开连接分配资源，抵御 SYN 洪水攻击
2. **连接数限制**：基于源 IP 限制并发连接数与新建连接速率，避免单 IP 耗尽所有连接资源
3. **静默丢弃**：`silent-drop`动作静默丢弃攻击连接，不返回任何数据包，避免攻击源感知防护策略
4. **连接陷阱**：`tarpit`动作保持攻击连接打开，占用攻击源的连接资源，抵御 CC 攻击

### 5.4.2 传输层 TCP 协议安全加固

通过 TCP 协议参数优化与异常报文过滤，加固传输层安全，抵御端口扫描、TCP 报文注入、异常连接等攻击，核心配置如下：

plaintext

```
global
    # 启用TCP窗口缩放，提升性能的同时避免异常窗口攻击
    tune.tcp.window_scaling on
    # 禁用TCP ECN，避免ECN滥用攻击
    tune.tcp.ecn off
frontend tcp_front
    bind 0.0.0.0:3306
    mode tcp
    # 异常TCP标志位过滤，抵御端口扫描与异常报文攻击
    acl invalid_tcp_flags tcp_flags FIN,SYN,RST,PSH,ACK,URG FIN,SYN
    acl port_scan tcp_flags FIN,SYN,RST,PSH,ACK,URG 0
    tcp-request content reject if invalid_tcp_flags || port_scan
    # 连接超时管控，避免半开连接占用资源
    timeout client 30s
    timeout connect 5s
```

### 5.4.3 应用层 HTTP 恶意请求防护

HAProxy 通过 ACL 规则与`http-request`动作，实现 HTTP 协议合规性校验、恶意请求拦截，抵御 SQL 注入、XSS、路径穿越、命令注入等常见 Web 攻击，生产级防护配置如下：

plaintext

```
frontend https_front
    bind 0.0.0.0:443 ssl crt /etc/haproxy/certs/www.example.com.pem
    mode http
    # 1. HTTP协议合规性校验
    acl invalid_method method !GET !POST !PUT !DELETE !OPTIONS !HEAD !PATCH
    acl invalid_http_version req_ver !1.0 !1.1 !2.0 !3.0
    acl empty_host hdr_len(Host) eq 0
    acl multiple_host hdr_cnt(Host) gt 1
    # 拦截不合规请求
    http-request deny status 400 if invalid_method || invalid_http_version || empty_host || multiple_host

    # 2. 恶意请求特征拦截
    acl sql_injection url -i -f /etc/haproxy/security/sql_injection.txt
    acl xss_attack url -i -f /etc/haproxy/security/xss.txt
    acl path_traversal url -i -f /etc/haproxy/security/path_traversal.txt
    acl command_injection url -i -f /etc/haproxy/security/command_injection.txt
    # 拦截恶意请求
    http-request deny status 403 if sql_injection || xss_attack || path_traversal || command_injection

    # 3. 爬虫与恶意User-Agent拦截
    acl malicious_ua user_agent -i -f /etc/haproxy/security/malicious_ua.txt
    acl empty_ua hdr_len(User-Agent) eq 0
    http-request deny status 403 if malicious_ua || empty_ua

    # 4. 请求体大小限制，避免大请求攻击
    acl large_request content_len gt 100M
    http-request deny status 413 if large_request
```

#### 核心防护规则说明

1. **协议合规性校验**：仅允许标准 HTTP 方法与协议版本，拦截畸形 HTTP 请求，避免协议解析漏洞攻击
2. **恶意特征匹配**：基于正则表达式匹配常见 Web 攻击的特征字符串，拦截注入类攻击
3. **爬虫防护**：拦截恶意爬虫 UA、空 UA，配合请求速率限制抵御 CC 攻击
4. **请求大小限制**：限制最大请求体大小，避免大文件上传攻击耗尽服务器资源

### 5.4.4 Web 安全响应头自动注入体系

通过自动注入 Web 安全响应头，强制客户端浏览器启用安全策略，抵御 XSS、点击劫持、MIME 类型嗅探等前端攻击，同时满足等保 2.0 合规要求，生产级标准配置如下：

plaintext

```
backend web_back
    mode http
    balance roundrobin
    # 全局安全响应头注入
    http-response set-header X-Frame-Options "DENY"
    http-response set-header X-Content-Type-Options "nosniff"
    http-response set-header X-XSS-Protection "1; mode=block"
    http-response set-header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'self'"
    http-response set-header Referrer-Policy "strict-origin-when-cross-origin"
    http-response set-header Permissions-Policy "geolocation=(), microphone=(), camera=()"
    # HSTS强制HTTPS，1年有效期，包含子域名，支持预加载
    http-response set-header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
    # 移除服务端版本信息，避免信息泄露
    http-response remove-header Server
    http-response remove-header X-Powered-By
    server web01 192.168.1.101:80 check
    server web02 192.168.1.102:80 check
```

### 5.4.5 精细化访问控制与黑白名单体系

基于 ACL 规则构建多层级访问控制体系，实现 IP 黑白名单、地理 IP 封禁、时间窗口访问控制、接口级权限管控，生产级配置如下：

plaintext

```
frontend https_front
    bind 0.0.0.0:443 ssl crt /etc/haproxy/certs/www.example.com.pem
    mode http
    # 1. 全局IP黑白名单
    acl trusted_ip src -f /etc/haproxy/security/whitelist_ip.txt
    acl black_ip src -f /etc/haproxy/security/blacklist_ip.txt
    http-request reject if black_ip

    # 2. 管理后台访问控制，仅允许信任IP访问
    acl admin_path path -beg /admin/ /system/ /actuator/
    http-request deny status 403 if admin_path !trusted_ip

    # 3. 时间窗口访问控制，非工作时间禁止管理后台访问
    acl working_hour day_of_week 1-5 hour 9-18
    http-request deny status 403 if admin_path !working_hour

    # 4. 接口级限流，非核心接口限制请求速率
    acl non_core_api path -beg /api/third/ /api/public/
    stick-table type ip size 100k expire 1m store http_req_rate(60s)
    http-request track-sc1 src if non_core_api
    acl api_rate_limit sc1_http_req_rate gt 60
    http-request deny status 429 if non_core_api api_rate_limit
```

## 5.5 SSL/TLS 性能优化学术级规范

TLS 加解密为 CPU 密集型操作，高并发场景下，TLS 卸载会成为 HAProxy 的性能瓶颈，本节基于密码学与计算机体系结构原理，定义学术级的性能优化规范，实现安全与性能的最优平衡。

### 5.5.1 硬件级加密加速方案

1. **AES-NI 指令集加速**
    
    - 原理：现代 Intel/AMD CPU 均内置 AES-NI 硬件加速指令集，可将 AES 对称加解密性能提升 10 倍以上，HAProxy 会自动检测并启用
    - 优化配置：确保操作系统启用 AES-NI 内核模块，HAProxy 编译时启用`USE_OPENSSL_AESNI`参数，生产环境默认启用
    
2. **Intel QAT 硬件加速卡**
    
    - 原理：Intel QuickAssist Technology 硬件加速卡可卸载 TLS 握手的非对称加密运算、对称加解密、哈希运算，将 CPU 占用降低 80% 以上
    - 适配配置：HAProxy 编译时启用`USE_QAT`参数，全局配置中启用 QAT 引擎：
        
        plaintext
        
        ```
        global
            ssl-engine qat algo ALL
        ```
        
    
3. **CPU 核心绑定优化**
    
    - 原理：TLS 运算的 CPU 上下文切换会导致显著的性能损耗，将 HAProxy 线程与物理 CPU 核心 1:1 绑定，避免跨 NUMA 节点调度
    - 优化配置：
        
        plaintext
        
        ```
        global
            nbthread 16
            cpu-map auto:1-16 0-15
        ```
        
    

### 5.5.2 加密套件与协议选型优化

1. **协议版本优化**：优先启用 TLS 1.3，握手延迟降低 50%，CPU 开销降低 30%，同时禁用所有低版本不安全协议
2. **加密套件优先级优化**：
    
    - 优先选择 ECDSA 证书 + AEAD 加密套件，非对称加密运算性能比 RSA 证书提升 3 倍以上
    - 优先选择 AES-GCM、ChaCha20-Poly1305 等 AEAD 认证加密套件，禁用 CBC、RC4 等弱套件
    - 密钥交换算法优先选择 ECDHE，禁用 DHE、RSA 静态密钥交换
    
3. **证书类型优化**：
    
    - 高并发场景优先使用 ECDSA P-256 证书，相比 RSA 2048 证书，握手性能提升 3 倍，证书体积更小
    - 兼容老旧客户端可同时配置 ECDSA 与 RSA 双证书，HAProxy 自动匹配客户端支持的证书类型
    

### 5.5.3 会话复用性能调优

1. **会话缓存优化**：单节点场景下，调整`tune.ssl.cachesize`与会话有效期，确保热点会话可被复用，推荐缓存大小为峰值并发连接数的 2 倍
2. **Session Ticket 优化**：集群场景下，使用统一的 Session Ticket 密钥，避免跨节点会话复用失败，密钥定期轮换，降低泄露风险
3. **TLS 1.3 0-RTT 优化**：对幂等请求启用 0-RTT，将握手延迟从 1-RTT 降低到 0-RTT，同时严格限制非幂等请求使用 0-RTT，避免重放攻击

### 5.5.4 进程 / 线程级 TLS 卸载调度优化

1. **单进程多线程模型优先**：TLS 卸载场景下，单进程多线程模型比多进程模型性能更高，线程间共享文件描述符与会话缓存，无进程间通信开销
2. **TLS 握手负载均衡**：启用`tune.ssl.force-private-cache`，每个线程拥有独立的会话缓存，避免多核锁竞争，高并发场景下性能提升 20% 以上
3. **SSL 记录层优化**：调整`tune.ssl.maxrecord`为 16384 字节，平衡 TCP 分片数量与加解密开销，大文件传输场景可适当调大，低延迟场景可适当调小

### 5.5.5 内核级网络参数优化

针对 TLS 流量特性，优化 Linux 内核网络参数，降低握手延迟与 CPU 开销，核心优化参数如下：

ini

```
# /etc/sysctl.conf 内核优化
# TCP SYN Cookie，抵御SYN洪水，提升握手成功率
net.ipv4.tcp_syncookies = 1
# TCP窗口缩放，提升大带宽场景性能
net.ipv4.tcp_window_scaling = 1
# 禁用TCP慢启动，提升长连接性能
net.ipv4.tcp_slow_start_after_idle = 0
# TCP拥塞控制算法，BBR算法显著提升TLS流量吞吐量
net.ipv4.tcp_congestion_control = bbr
# 本地端口范围，提升最大并发连接数
net.ipv4.ip_local_port_range = 1024 65535
# TIME_WAIT套接字重用，避免端口耗尽
net.ipv4.tcp_tw_reuse = 1
# TCP最大半开连接数，抵御SYN洪水
net.ipv4.tcp_max_syn_backlog = 65536
# 套接字接收/发送缓冲区，优化TLS大流量传输
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
```

## 5.6 合规性配置基线

### 5.6.1 等保 2.0 三级合规配置

针对网络安全等级保护 2.0 三级要求，核心合规配置基线如下：

1. **传输加密要求**：禁用 SSL 3.0、TLS 1.0、TLS 1.1，仅启用 TLS 1.2+，使用强加密套件
2. **访问控制要求**：管理后台 IP 白名单限制，接口级访问控制，操作日志全量记录
3. **安全审计要求**：全量访问日志开启，包含源 IP、请求时间、请求方法、URL、响应状态码、响应时长、用户代理等字段，日志留存不少于 6 个月
4. **恶意代码防护**：启用 SQL 注入、XSS、路径穿越等 Web 攻击防护，恶意请求拦截与日志记录
5. **安全加固要求**：自动注入 Web 安全响应头，HSTS 强制 HTTPS，隐藏服务端版本信息

### 5.6.2 PCI-DSS 支付行业合规配置

针对 PCI-DSS 支付卡行业数据安全标准，核心合规配置基线如下：

1. **TLS 协议要求**：强制禁用 TLS 1.0 及以下版本，仅启用 TLS 1.2+，优先 TLS 1.3
2. **加密套件要求**：仅启用 PCI-DSS 认可的强加密套件，禁用所有弱加密套件与算法
3. **mTLS 双向认证**：支付接口必须启用 mTLS 双向认证，严格校验客户端证书身份
4. **OCSP Stapling**：必须启用 OCSP Stapling，确保证书吊销状态实时校验
5. **日志审计要求**：全量交易请求日志全字段记录，日志留存不少于 1 年，不可篡改
6. **安全头要求**：强制启用 HSTS、CSP、X-Frame-Options 等安全响应头，禁止明文 HTTP 传输

### 5.6.3 通用隐私合规配置要求

针对 GDPR、个人信息保护法等隐私合规要求，核心配置基线如下：

1. **全链路加密**：端到端 TLS 加密，无明文传输个人敏感信息
2. **OCSP Stapling**：启用 OCSP Stapling，避免客户端 OCSP 查询泄露用户隐私
3. **会话数据管控**：禁用 TLS Session Ticket，或定期轮换 Ticket 密钥，会话数据定期过期
4. **访问日志脱敏**：访问日志中对身份证号、手机号、银行卡号等敏感信息脱敏处理
5. **Cookie 安全**：所有 Cookie 必须标记 Secure、HttpOnly、SameSite 属性，禁止第三方 Cookie 追踪

## 5.7 生产级完整配置模板

### 5.7.1 通用 HTTPS 站点标准配置

plaintext

```
# 全局配置
global
    daemon
    master-worker
    nbthread 8
    cpu-map auto:1-8 0-7
    maxconn 100000
    log /dev/log local0 info
    stats socket /var/run/haproxy.sock mode 660 level admin
    pidfile /var/run/haproxy.pid
    user haproxy
    group haproxy

    # SSL全局安全基线
    ssl-default-bind-options no-sslv3 no-tlsv10 no-tlsv11 no-tls-tickets ssl-prefer-server-ciphers
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305
    ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
    ssl-default-server-options no-sslv3 no-tlsv10 no-tlsv11
    ssl-default-server-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384
    ssl-default-server-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
    tune.ssl.default-dh-param 2048
    tune.ssl.cachesize 1000000
    tune.ssl.lifetime 1800
    tune.ssl.maxrecord 16384

# 默认配置
defaults
    mode http
    log global
    option httplog
    option forwardfor
    option http-server-close
    option redispatch
    retries 3
    retry-on conn-failure timeout 500 empty-response invalid-response
    timeout connect 5s
    timeout client 30s
    timeout server 30s
    timeout http-request 10s
    timeout http-keep-alive 10s
    timeout queue 30s
    timeout check 2s
    balance roundrobin
    inter 2s
    rise 2
    fall 3

# HTTP 80端口跳转HTTPS
frontend http_front
    bind 0.0.0.0:80
    mode http
    acl acme_challenge path_beg /.well-known/acme-challenge/
    use_backend acme_back if acme_challenge
    http-request redirect scheme https code 301 if !acme_challenge
    default_backend web_back

# HTTPS 443端口主入口
frontend https_front
    bind 0.0.0.0:443 ssl \
        crt /etc/haproxy/certs/ \
        crt /etc/haproxy/certs/default.example.com.pem \
        alpn h2,http/1.1 \
        ocsp-update on \
        ocsp-strict on \
        ca-file /etc/haproxy/certs/root_ca.pem \
        ssl-min-ver TLSv1.2 \
        ssl-max-ver TLSv1.3
    mode http

    # 基础安全防护
    stick-table type ip size 200k expire 10m store conn_cur,conn_rate(10s),http_req_rate(10s)
    http-request track-sc0 src
    acl too_many_conn sc0_conn_cur gt 50
    acl high_req_rate sc0_http_req_rate gt 200
    http-request deny status 429 if too_many_conn
    http-request tarpit if high_req_rate

    # 协议合规性校验
    acl invalid_method method !GET !POST !PUT !DELETE !OPTIONS !HEAD !PATCH
    acl empty_host hdr_len(Host) eq 0
    http-request deny status 400 if invalid_method || empty_host

    # 多站点路由
    acl site_www ssl_fc_sni -i www.example.com example.com
    acl site_api ssl_fc_sni -i api.example.com
    use_backend www_back if site_www
    use_backend api_back if site_api
    default_backend www_back

# 主站后端
backend www_back
    mode http
    option httpchk GET /health HTTP/1.1
    http-check send hdr Host www.example.com
    http-check expect status 200
    observe layer7
    on-error fail-check
    error-limit 5
    on-marked-down shutdown-sessions

    # Web安全响应头
    http-response set-header X-Frame-Options "DENY"
    http-response set-header X-Content-Type-Options "nosniff"
    http-response set-header X-XSS-Protection "1; mode=block"
    http-response set-header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'"
    http-response set-header Referrer-Policy "strict-origin-when-cross-origin"
    http-response set-header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
    http-response remove-header Server
    http-response remove-header X-Powered-By

    cookie SERVERID insert nocache indirect secure httponly maxidle 3600s
    server web01 192.168.1.101:80 check cookie web01 slowstart 60s
    server web02 192.168.1.102:80 check cookie web02 slowstart 60s

# API后端
backend api_back
    mode http
    option httpchk GET /health HTTP/1.1
    http-check send hdr Host api.example.com
    http-check expect status 200
    server api01 192.168.1.151:8080 check
    server api02 192.168.1.152:8080 check

# ACME证书验证后端
backend acme_back
    mode http
    server acme 127.0.0.1:8080 check
```

### 5.7.2 高安全级 mTLS 双向认证配置

plaintext

```
frontend mtls_api_front
    bind 0.0.0.0:443 ssl \
        crt /etc/haproxy/certs/api.example.com.pem \
        ca-file /etc/haproxy/certs/client_ca.pem \
        crl-file /etc/haproxy/certs/client_crl.pem \
        verify required \
        alpn h2,http/1.1 \
        ssl-min-ver TLSv1.2 \
        strict-sni
    mode http

    # 证书有效性校验
    acl cert_valid ssl_c_verify_result eq 0
    acl cert_not_expired ssl_c_notafter gt date
    http-request deny status 403 if !cert_valid || !cert_not_expired

    # 基于客户端证书CN的访问控制
    acl admin_cert ssl_c_s_dn(CN) -i admin-api
    acl service_cert ssl_c_s_dn(CN) -i service-api
    acl readonly_cert ssl_c_s_dn(CN) -i readonly-api

    # 接口权限管控
    acl write_method method POST PUT DELETE PATCH
    http-request deny status 403 if readonly_cert write_method

    use_backend admin_api_back if admin_cert
    use_backend service_api_back if service_cert
    use_backend readonly_api_back if readonly_cert
    default_backend readonly_api_back

backend admin_api_back
    mode http
    balance roundrobin
    option httpchk GET /health HTTP/1.1
    http-check send hdr Host api.example.com
    http-check expect status 200
    # 端到端TLS加密
    server api01 192.168.1.151:8443 check ssl verify required ca-file /etc/haproxy/certs/upstream_ca.pem
    server api02 192.168.1.152:8443 check ssl verify required ca-file /etc/haproxy/certs/upstream_ca.pem
```

### 5.7.3 四层 TLS 透传 SNI 路由配置

plaintext

```
global
    daemon
    master-worker
    nbthread 4
    maxconn 50000
    log /dev/log local0 info
    stats socket /var/run/haproxy.sock mode 660 level admin
    pidfile /var/run/haproxy.pid
    user haproxy
    group haproxy

defaults
    mode tcp
    log global
    option tcplog
    option redispatch
    retries 2
    timeout connect 3s
    timeout client 1800s
    timeout server 1800s
    timeout check 2s
    balance leastconn
    inter 2s
    rise 2
    fall 3

frontend tls_passthrough_front
    bind 0.0.0.0:443
    mode tcp
    # 提取SNI字段
    tcp-request inspect-delay 5s
    tcp-request content accept if { req_ssl_hello_type 1 }

    # 源IP访问控制
    acl trusted_ip src 10.0.0.0/8 192.168.0.0/16
    tcp-request content reject if !trusted_ip

    # SNI路由
    acl site_www req_ssl_sni -i www.example.com
    acl site_api req_ssl_sni -i api.example.com
    use_backend www_tls_back if site_www
    use_backend api_tls_back if site_api
    default_backend default_tls_back

backend www_tls_back
    mode tcp
    server web01 192.168.1.101:443 check ssl verify none
    server web02 192.168.1.102:443 check ssl verify none

backend api_tls_back
    mode tcp
    server api01 192.168.1.151:443 check ssl verify none
    server api02 192.168.1.152:443 check ssl verify none

backend default_tls_back
    mode tcp
    server default 192.168.1.200:443 check ssl verify none
```

继续下一章