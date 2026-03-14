#### 核心标准答案

HTTPS 反向代理的核心，是在 Nginx 上配置 SSL 证书，完成 HTTPS 握手与加解密，再将请求转发到后端服务，核心配置步骤与最小可用模板如下：

1. 准备合法的 SSL 证书：服务器证书文件（.crt/.cer）、证书私钥文件（.key），上传到 Nginx 服务器的指定目录（如`/etc/nginx/ssl/`）。
2. 编辑 Nginx 配置文件，在`http`块下配置`server`节点，监听 443 端口并开启 SSL，配置证书路径、反向代理规则。
3. 配置完成后，执行`nginx -t`校验配置语法，执行`nginx -s reload`重载配置生效。

**最小可用配置模板**

nginx

```
server {
    # 监听443端口，开启SSL
    listen 443 ssl;
    # 绑定你的域名
    server_name your-domain.com;

    # SSL证书配置
    ssl_certificate /etc/nginx/ssl/your-domain.crt;  # 证书文件路径
    ssl_certificate_key /etc/nginx/ssl/your-domain.key; # 私钥文件路径

    # 反向代理核心配置
    location / {
        # 后端服务地址（HTTP/HTTPS均可）
        proxy_pass http://127.0.0.1:8080;
        # 传递客户端真实IP、Host等核心头信息
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# HTTP强制跳转到HTTPS（可选，推荐配置）
server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$server_name$request_uri;
}
```

#### 专家级拓展

- 安全优化配置（生产环境必加）：
    
    nginx
    
    ```
    # 禁用不安全的SSL协议，仅支持TLS1.2+
    ssl_protocols TLSv1.2 TLSv1.3;
    # 配置安全加密套件
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers on;
    # 会话缓存优化，减少重复握手
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    # 开启HSTS，强制浏览器使用HTTPS访问
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    ```
    
- 核心原理：Nginx 作为 SSL 终结点，客户端与 Nginx 建立 HTTPS 连接，Nginx 完成加解密后，将明文请求转发给后端服务，降低后端服务的 SSL 性能开销。
- 故障排查工具：运维场景下，常用`openssl s_client -connect your-domain.com:443`排查证书有效性、握手问题，用`curl -v https://your-domain.com`排查代理转发、头信息问题。

#### 面试避坑指南

- 严禁只说配置步骤，不给可落地的配置模板，会被判定为无实际操作经验；
- 避免遗漏`proxy_set_header`核心头配置，生产环境会导致后端无法获取客户端真实 IP、出现跨域问题；
- 严禁配置错误的证书路径，证书文件必须对 Nginx 进程有读权限，否则会启动失败。