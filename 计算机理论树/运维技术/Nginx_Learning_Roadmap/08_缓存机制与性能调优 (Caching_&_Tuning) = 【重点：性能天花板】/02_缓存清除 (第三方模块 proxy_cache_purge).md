
## 02 缓存清除

Nginx 开源版原生仅支持**被动缓存清除**（通过`inactive`过期、`max_size`LRU 淘汰、手动删除磁盘文件），无原生主动缓存清除能力；Nginx Plus 商业版提供原生`proxy_cache_purge`主动清除功能，开源版需通过第三方模块**ngx_cache_purge**实现主动精准清除缓存，是生产环境缓存内容更新的核心解决方案。

### 2.1 被动缓存清除方案

无需额外模块，适用于非紧急的缓存更新场景，是 Nginx 原生支持的清除方式：

1. **磁盘文件手动删除**：通过`find`命令精准删除指定缓存文件，示例：
    
    bash
    
    运行
    
    ```
    # 删除首页对应的缓存文件
    find /data/nginx/cache -type f -exec grep -l "GET www.example.com/index.html" {} \; -delete
    # 清空所有缓存
    rm -rf /data/nginx/cache/*
    ```
    
2. **调整缓存有效期**：临时缩短`proxy_cache_valid`与`inactive`时间，触发过期缓存自动清理，更新完成后恢复原配置。
3. **修改缓存键**：通过调整`proxy_cache_key`规则，生成全新的缓存索引，原有缓存内容会因无命中被`inactive`机制自动淘汰。

### 2.2 主动缓存清除：ngx_cache_purge 第三方模块

#### 2.2.1 模块安装

该模块需在 Nginx 源码编译时通过`--add-module`参数引入，步骤如下：

bash

运行

```
# 1. 下载模块源码
wget https://github.com/FRiCKLE/ngx_cache_purge/archive/refs/heads/master.zip
unzip master.zip
# 2. 进入Nginx源码目录，新增模块编译
cd nginx-1.24.0
./configure --with-http_ssl_module --with-http_v2_module --add-module=../ngx_cache_purge-master
# 3. 平滑升级Nginx，不中断服务
make && make upgrade
```

#### 2.2.2 核心指令与语法

**核心指令**：`proxy_cache_purge`

**官方语法**：`proxy_cache_purge zone_name key;`

**合法作用域**：`location`

**核心功能**：基于指定的缓存键，精准清除共享内存索引与对应的磁盘缓存文件。

#### 2.2.3 标准安全配置示例

nginx

```
http {
    # 缓存全局定义，与01节配置一致
    proxy_cache_path /data/nginx/cache
        levels=1:2
        use_temp_path=off
        keys_zone=WEB_CACHE:100m
        inactive=7d
        max_size=200g;

    server {
        listen 443 ssl http2;
        server_name www.example.com;
        ssl_certificate /etc/nginx/ssl/example.com.crt;
        ssl_certificate_key /etc/nginx/ssl/example.com.key;

        # 缓存清除专用接口，必须做严格的安全限制
        location ~ /purge(/.*) {
            # 1. 严格IP白名单，仅允许内网、运维办公网IP访问
            allow 192.168.1.0/24;
            allow 223.xxx.xxx.xxx/32;
            deny all;

            # 2. 基础身份认证，双重防护
            auth_basic "Cache Purge - Authorized Access Only";
            auth_basic_user_file /etc/nginx/auth_basic/purge.htpasswd;

            # 3. 核心清除配置，引用缓存区与缓存键
            proxy_cache_purge WEB_CACHE $scheme$host$1$is_args$args;

            # 4. 清除结果响应头
            add_header X-Cache-Purged-Key $scheme$host$1$is_args$args always;
        }

        # 业务缓存配置，与01节一致，此处省略
    }
}
```

#### 2.2.4 清除使用方式

配置完成后，可通过 HTTP 请求精准清除指定缓存，示例：

bash

运行

```
# 清除首页缓存
curl -u admin:your_password https://www.example.com/purge/index.html
# 清除指定图片缓存
curl -u admin:your_password https://www.example.com/purge/static/images/logo.png
# 通配符清除（需模块开启正则支持）
curl -u admin:your_password https://www.example.com/purge/static/css/*
```

### 2.3 关键安全注意事项

1. **清除接口必须做多层安全防护**：缓存清除接口直接操作服务端缓存，若被恶意访问，会导致缓存被完全清空，源站瞬间被打垮。必须配置「IP 白名单 + HTTP 基础认证」双重防护，高安全场景需额外配置客户端证书认证。
2. **禁止公网全量开放清除接口**：严禁将清除接口配置为无限制公网访问，生产环境必须仅对运维内网 IP 开放。
3. **清除操作灰度执行**：大规模缓存清除前，先单条测试验证，再分批执行，避免一次性清空全量缓存导致源站压力突增，引发服务雪崩。
4. **操作日志审计**：开启清除接口的访问日志，记录所有清除操作的 IP、账号、清除路径，便于安全审计与故障追溯。

---
