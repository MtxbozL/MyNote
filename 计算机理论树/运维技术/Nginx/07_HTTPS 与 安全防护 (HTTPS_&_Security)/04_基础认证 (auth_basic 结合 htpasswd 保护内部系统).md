
Nginx 的 HTTP 基础身份认证能力由**ngx_http_auth_basic_module**标准模块提供，基于 HTTP Basic Authentication 协议实现，通过用户名 + 密码的方式对访问请求进行身份校验，适用于内部系统、测试环境、管理后台等非公开访问场景的轻量级身份认证。

### 4.1 核心指令与语法

| 指令                     | 官方语法                         | 合法作用域                                     | 核心功能                                 |
| ---------------------- | ---------------------------- | ----------------------------------------- | ------------------------------------ |
| `auth_basic`           | `auth_basic string off;`     | `http`、`server`、`location`、`limit_except` | 启用 HTTP 基础认证，配置认证弹窗的提示字符串；`off`为关闭认证 |
| `auth_basic_user_file` | `auth_basic_user_file file;` | `http`、`server`、`location`、`limit_except` | 指定存储用户名与加密密码的文件路径                    |

### 4.2 密码文件生成与格式

密码文件需通过`htpasswd`工具生成，该工具由`httpd-tools`软件包提供，需提前安装：

```bash
# CentOS/RHEL 安装
yum install -y httpd-tools
# Debian/Ubuntu 安装
apt install -y apache2-utils
```

#### 4.2.1 密码文件生成命令

```bash
# 生成新的密码文件，添加用户admin，执行后输入密码
htpasswd -c /etc/nginx/auth_basic/.htpasswd admin
# 向已存在的密码文件中追加新用户test
htpasswd /etc/nginx/auth_basic/.htpasswd test
# 删除指定用户
htpasswd -D /etc/nginx/auth_basic/.htpasswd test
```

#### 4.2.2 密码文件格式

密码文件每行对应一个用户，格式为`用户名:加密密码`，支持多种加密算法，默认使用 APR1 MD5 加密，禁止使用明文存储。

### 4.3 标准配置示例

#### 4.3.1 全站基础认证

```nginx
server {
    listen 443 ssl http2;
    server_name test.example.com;
    root /var/www/test;

    # 启用基础认证
    auth_basic "Test Environment - Authorized Access Only";
    # 指定密码文件路径
    auth_basic_user_file /etc/nginx/auth_basic/.htpasswd;
}
```

#### 4.3.2 特定路径认证

```nginx
server {
    listen 443 ssl http2;
    server_name www.example.com;
    root /var/www/html;

    # 公开路径无需认证
    location / {
        index index.html;
    }

    # 管理后台路径启用认证
    location /admin/ {
        auth_basic "Admin System - Login Required";
        auth_basic_user_file /etc/nginx/auth_basic/admin.htpasswd;
        proxy_pass http://admin_backend;
    }

    # 内部接口启用认证，同时配合IP白名单
    location /api/internal/ {
        # IP白名单优先，内网IP无需认证
        satisfy any;
        allow 192.168.1.0/24;
        deny all;

        # 外网IP需通过账号密码认证
        auth_basic "Internal API - Authorized Access Only";
        auth_basic_user_file /etc/nginx/auth_basic/api.htpasswd;

        proxy_pass http://api_backend;
    }
}
```

> 关键配置说明：`satisfy any`指令表示满足「IP 白名单」或「认证通过」任意一个条件即可访问；`satisfy all`（默认）表示必须同时满足所有条件。

### 4.4 关键注意事项

1. **必须配合 HTTPS 使用**：HTTP Basic Authentication 协议的用户名与密码仅通过 Base64 编码传输，无加密，HTTP 明文环境下极易被窃听，必须在 HTTPS 协议下使用。
2. **密码文件权限控制**：密码文件必须设置为`600`权限，所有者为 Nginx 运行用户，禁止全局可读，同时需将密码文件放在 Web 根目录之外，防止被外部下载。
3. **密码强度要求**：必须为用户配置高强度密码，避免弱密码被暴力破解，可配合限流模块限制认证失败的请求速率。
4. **浏览器兼容性**：认证提示字符串建议使用英文，部分旧版浏览器对中文提示字符串存在乱码问题。
5. **登出机制限制**：HTTP Basic Authentication 协议无原生登出机制，浏览器关闭前会一直缓存认证信息，仅适用于内部可信场景，不建议用于面向公网的用户系统。

---
