## 一、核心定义

OCSP 装订（正式名称为 **TLS Certificate Status Request 扩展**），是 TLS 协议的安全扩展，核心是**把 SSL 证书吊销状态的校验工作，从客户端转移到服务端**：服务端提前向 CA（证书颁发机构）的 OCSP 服务器获取证书吊销状态的签名结果，在 TLS 握手时和证书链一起 “装订” 发送给客户端，客户端无需再单独访问 CA 的 OCSP 服务器，即可完成证书有效性校验。

### 前置基础：什么是 OCSP？

OCSP（Online Certificate Status Protocol，在线证书状态协议），是 PKI 公钥体系中用于**实时验证 SSL 证书是否被吊销**的核心协议，用来替代传统的 CRL（证书吊销列表）。

- 传统 CRL：需要客户端下载完整的证书吊销列表，文件体积大、更新周期长，实时性差；
- OCSP：客户端单独查询单张证书的状态，仅传输极小的查询 / 响应包，实时性更强，但存在原生痛点。

---

## 二、传统 OCSP 的核心痛点（OCSP 装订要解决的问题）

传统 OCSP 由客户端（浏览器）主动向 CA 的 OCSP 服务器发起查询，存在 4 个致命问题：

1. **用户隐私泄露**：客户端每次 HTTPS 握手都要访问 CA 的 OCSP 服务器，CA 可以精准获取「哪个用户在访问哪个网站」，泄露用户的访问行为和隐私。
2. **HTTPS 性能损耗**：客户端需要额外发起一次到 CA 服务器的 HTTP 请求，增加 1-2 次 RTT 网络往返，大幅延长 TLS 握手时间；尤其是跨地域访问（比如国内用户访问海外 CA 的 OCSP 服务器），延迟会达到数百毫秒，严重影响页面加载速度。
3. **服务可用性风险**：如果 CA 的 OCSP 服务器故障、被网络拦截、访问拥堵，客户端无法完成证书校验，可能会直接拒绝连接，导致网站无法访问；或跳过校验，带来中间人攻击的安全风险。
4. **CA 服务器压力过大**：海量客户端都向 CA 发起 OCSP 查询，给 CA 的服务带来巨大的带宽和请求压力，极端情况会导致 OCSP 服务响应超时，影响全网站点的 HTTPS 访问。

---

## 三、OCSP 装订完整工作流程

1. **服务端预查询**：Nginx 等服务端提前向 CA 的 OCSP 服务器发起查询请求，获取当前站点证书的吊销状态响应。该响应由 CA 的私钥签名，无法篡改，自带有效期（通常 3-7 天）。
2. **服务端缓存**：Nginx 将合法的 OCSP 响应缓存起来，在有效期内可重复给所有客户端使用，无需频繁向 CA 发起查询。
3. **客户端握手协商**：用户浏览器发起 HTTPS 请求，在 TLS 握手的`Client Hello`阶段，会携带`status_request`扩展，告知服务端自身支持 OCSP 装订。
4. **服务端装订响应**：服务端收到握手请求后，在`Server Hello`阶段，把缓存的、CA 签名的 OCSP 响应，和站点证书链一起发送给客户端。
5. **客户端校验完成**：客户端收到 OCSP 响应后，先验证 CA 的签名合法性，确认响应未被篡改，再读取证书状态：
    
    - 证书有效：完成后续 TLS 握手，建立加密连接；
    - 证书已吊销：直接终止连接，向用户提示安全风险。
    

---

## 四、OCSP 装订的核心优势

1. **彻底保护用户隐私**：客户端完全不需要访问 CA 的 OCSP 服务器，CA 无法获取用户的访问行为，从根源消除隐私泄露风险。
2. **大幅提升 HTTPS 性能**：消除了客户端到 CA 服务器的额外 RTT，TLS 握手时间大幅缩短，页面加载速度显著提升，弱网、跨地域场景效果尤为明显。
3. **提升服务可用性**：OCSP 查询由服务端完成，可提前缓存、重试、多线路优化；即使 CA 的 OCSP 服务器临时不可用，服务端仍可使用缓存的有效 OCSP 响应，不会影响用户正常访问。
4. **降低 CA 服务压力**：所有用户的 OCSP 查询由服务端统一完成，缓存后可被海量用户复用，CA 的请求量会降低几个数量级，也减少了因 OCSP 服务拥堵导致的超时问题。
5. **安全性更强**：OCSP 响应由 CA 私钥签名，无法被篡改；服务端可确保 OCSP 响应的有效性，避免客户端因网络问题跳过证书吊销校验，降低中间人攻击风险。

---

## 五、Nginx 中 OCSP 装订的完整可落地配置

### 前置要求

- Nginx 版本≥1.3.7（主流发行版均满足），编译时开启`--with-http_ssl_module`（默认已开启）；
- 证书文件必须包含**完整证书链**（站点证书 + 中间证书），用于 OCSP 响应的签名校验；
- 服务器防火墙放行出站 80/443 端口，可正常访问 CA 的 OCSP 服务器；
- 配置稳定可用的 DNS 服务器，用于解析 CA 的 OCSP 域名。

### 完整配置示例

nginx

```
server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    # 基础SSL证书配置
    ssl_certificate /etc/nginx/ssl/example.com_fullchain.pem; # 完整证书链（站点证书+中间证书）
    ssl_certificate_key /etc/nginx/ssl/example.com.key; # 证书私钥

    # ========== OCSP装订核心配置 ==========
    # 开启OCSP装订功能
    ssl_stapling on;
    # 开启OCSP响应合法性校验，确保响应来自可信CA
    ssl_stapling_verify on;
    # 信任的根证书+中间证书链，用于校验OCSP响应的签名，可与ssl_certificate共用同一文件
    ssl_trusted_certificate /etc/nginx/ssl/example.com_fullchain.pem;
    # 配置DNS服务器，用于解析CA的OCSP域名，国内推荐稳定公共DNS
    resolver 223.5.5.5 119.29.29.29 114.114.114.114 valid=300s;
    # DNS解析超时时间
    resolver_timeout 5s;

    # ========== 配套SSL优化配置（推荐开启） ==========
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL_SESSION:10m;
    ssl_session_timeout 1d;

    # 站点基础配置
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

配置完成后，先执行`nginx -t`校验语法，确认无误后执行`nginx -s reload`平滑重载配置。

---

## 六、生效验证方法

### 1. 命令行验证（最准确，推荐）

bash

运行

```
openssl s_client -connect example.com:443 -status -tlsextdebug < /dev/null 2>&1 | grep "OCSP Response Status"
```

- 生效成功：返回`OCSP Response Status: successful (0x0)`；
- 未生效：无相关 OCSP 响应输出，需排查配置问题。

### 2. 在线工具验证

使用 SSL Labs Server Test（[https://www.ssllabs.com/ssltest/](https://www.ssllabs.com/ssltest/)），输入域名完成检测后，在「Protocol Details」板块查看「OCSP Stapling」是否为`Yes`。

### 3. 浏览器验证

Chrome 浏览器访问站点，按 F12 打开开发者工具，进入「安全」板块，查看证书详情，可看到 OCSP 装订的启用状态。

---

## 七、常见踩坑与避坑指南

1. **证书链不完整，OCSP 查询失败**
    
    - 问题：`ssl_trusted_certificate`未配置完整的根证书 + 中间证书，Nginx 无法校验 OCSP 响应的签名，无法获取有效响应。
    - 解决：确保证书文件包含完整的证书链，`ssl_certificate`和`ssl_trusted_certificate`使用包含完整链的 pem 文件。
    
2. **网络 / 防火墙拦截，无法访问 OCSP 服务器**
    
    - 问题：服务器防火墙拦截了出站 80/443 端口，或 CA 的 OCSP 服务器被网络拦截，Nginx 无法发起 OCSP 查询。
    - 解决：放行服务器出站的 80/443 端口，测试服务器能否正常访问 CA 的 OCSP 域名，必要时配置代理。
    
3. **未配置 DNS 服务器，无法解析 OCSP 域名**
    
    - 问题：未配置`resolver`指令，或 DNS 服务器不可用，Nginx 无法解析 CA 的 OCSP 服务器域名，导致查询失败。
    - 解决：配置稳定可用的公共 DNS 服务器，确保服务器能正常解析域名。
    
4. **证书无 OCSP 地址，无法发起查询**
    
    - 问题：自签名证书、老旧证书未包含 OCSP 服务器地址，Nginx 无法发起查询。
    - 解决：使用正规 CA 签发的证书，可通过以下命令查看证书的 OCSP 地址：
        
        bash
        
        运行
        
        ```
        openssl x509 -in 证书文件.pem -noout -ocsp_uri
        ```
        
    
5. **兼容性问题**
    
    所有现代浏览器（Chrome、Firefox、Edge、Safari）均支持 OCSP 装订；老旧浏览器（如 IE6）不支持，但不会影响 TLS 握手，会自动回退到传统 OCSP 查询，无兼容性风险。