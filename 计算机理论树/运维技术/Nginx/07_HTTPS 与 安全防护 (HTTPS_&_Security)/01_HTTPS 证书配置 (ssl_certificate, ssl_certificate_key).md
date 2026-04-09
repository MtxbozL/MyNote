
本章聚焦 Nginx 的传输层加密通信与应用层全维度防护能力，覆盖 HTTPS 协议的标准化配置与性能优化、访问权限管控、身份认证、资源防盗链、服务信息泄露防护，以及高并发场景下的限流防攻击核心机制。所有内容严格遵循 Nginx 官方规范与行业安全最佳实践，是企业级 Nginx 服务实现安全合规、稳定运行的核心技术支撑。

本章所有核心指令均来自 Nginx 标准内置模块，除特殊说明外，无需额外第三方模块与源码编译配置，仅需确保编译时启用`--with-http_ssl_module`（HTTPS 核心模块）即可。

---

### 1.1 模块与协议概述

HTTPS 的核心是基于 TLS/SSL 协议实现 HTTP 传输的端到端加密，Nginx 的 HTTPS 能力由**ngx_http_ssl_module**标准模块提供，源码编译时需通过`--with-http_ssl_module`参数启用，yum/apt 等包管理器安装的 Nginx 默认已启用该模块。

TLS 协议的核心作用：

1. 身份认证：通过数字证书验证服务端身份，防止中间人攻击；
2. 数据加密：对传输层数据进行对称加密，防止明文传输被窃听；
3. 完整性校验：通过哈希算法验证传输数据未被篡改。

### 1.2 核心标准化指令

#### 1.2.1 证书与私钥核心指令

|指令|官方语法|合法作用域|核心功能|
|---|---|---|---|
|`ssl_certificate`|`ssl_certificate file;`|`http`、`server`|指定服务端数字证书文件路径，支持 PEM 格式，可包含证书链|
|`ssl_certificate_key`|`ssl_certificate_key file;`|`http`、`server`|指定证书对应的私钥文件路径，支持 PEM 格式，需与证书公钥配对|

#### 1.2.2 配套基础指令

HTTPS 配置必须配合 TLS 协议版本与加密套件指令，实现合规性与兼容性的平衡：

|指令|核心语法|功能说明|
|---|---|---|
|`ssl_protocols`|`ssl_protocols TLSv1.2 TLSv1.3;`|指定启用的 TLS 协议版本，禁用不安全的 SSLv2、SSLv3、TLSv1.0、TLSv1.1|
|`ssl_ciphers`|`ssl_ciphers [cipher_list];`|指定启用的加密套件列表，遵循 OpenSSL 语法规范，禁用弱加密算法|
|`ssl_prefer_server_ciphers`|`ssl_prefer_server_ciphers on\|off;`|强制优先使用服务端配置的加密套件，而非客户端提交的套件列表，防止弱加密降级攻击|

### 1.3 数字证书核心分类

|证书类型|简称|身份验证等级|适用场景|
|---|---|---|---|
|域名验证型证书|DV|低，仅验证域名所有权|个人网站、小型博客、非交易类站点|
|组织验证型证书|OV|中，验证域名所有权 + 企业主体合法性|企业官网、中小型业务系统|
|扩展验证型证书|EV|高，严格验证企业主体资质与法律实体|金融、支付、电商等强合规场景|

按域名覆盖范围可分为：单域名证书、泛域名证书（`*.example.com`）、多域名证书（SAN 证书），需根据业务域名场景选择对应证书类型。

### 1.4 标准合规配置示例

```nginx
# http全局块基础配置，可被server块继承
http {
    # 全局TLS基础配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers on;

    # 业务HTTPS server块
    server {
        # 监听443端口，启用ssl，同时开启HTTP/2
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name example.com www.example.com;

        # 核心证书与私钥配置
        ssl_certificate /etc/nginx/ssl/example.com_fullchain.pem;
        ssl_certificate_key /etc/nginx/ssl/example.com_privkey.pem;

        # 业务配置
        root /var/www/example.com;
        index index.html;
    }
}
```

### 1.5 配置关键注意事项

1. **证书文件规范**：`ssl_certificate`必须配置完整证书链（fullchain），包含域名证书 + 中间 CA 证书 + 根证书，仅配置域名证书会导致移动端、旧版浏览器出现证书不可信错误。
2. **私钥文件权限**：私钥文件必须严格控制权限，建议设置为`600`，所有者为 Nginx 运行用户，禁止全局可读，防止私钥泄露。
3. **证书与私钥配对**：证书公钥与私钥必须严格配对，可通过 OpenSSL 命令校验一致性，配置错误会导致 Nginx 启动失败。
    
    ```bash
    # 校验证书与私钥的模数是否一致，一致则配对成功
    openssl x509 -noout -modulus -in example.com_fullchain.pem | openssl md5
    openssl rsa -noout -modulus -in example.com_privkey.pem | openssl md5
    ```
    
4. **证书有效期管理**：需提前配置证书到期提醒，DV 证书默认有效期 90 天，OV/EV 证书默认有效期 1 年，证书过期会导致所有客户端无法访问。
5. **作用域优先级**：`server`块内的证书配置会覆盖`http`全局块的配置，多域名 HTTPS 站点需为每个 server 块单独配置对应域名的证书。

---
