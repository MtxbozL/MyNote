这是一个**生产环境常用的 Nginx 一键部署 Shell 脚本**，包含编译安装、系统服务配置、防火墙设置及基础反向代理配置模板，适合 CentOS/Ubuntu 系统，可直接用于面试或实际部署。

### Nginx 部署脚本 (`deploy_nginx.sh`)

```Bash
#!/bin/bash
set -e

# ==========================================
# 通用配置
# ==========================================
NGINX_VERSION="1.24.0"
INSTALL_PREFIX="/usr/local/nginx"
CONFIG_PATH="${INSTALL_PREFIX}/conf/nginx.conf"
PID_PATH="/var/run/nginx.pid"
TEMP_DIR="/tmp/nginx_install"

# 颜色输出
COLOR_GREEN="\033[32m"
COLOR_RED="\033[31m"
COLOR_YELLOW="\033[33m"
COLOR_RESET="\033[0m"

# ==========================================
# 工具函数
# ==========================================
log_info() { echo -e "${COLOR_GREEN}[INFO] $1${COLOR_RESET}"; }
log_warn() { echo -e "${COLOR_YELLOW}[WARN] $1${COLOR_RESET}"; }
log_error() { echo -e "${COLOR_RED}[ERROR] $1${COLOR_RESET}"; exit 1; }

# 检查命令是否存在，不存在则尝试安装
check_and_install() {
    local cmd=$1
    local pkg=$2
    if ! command -v "$cmd" &> /dev/null; then
        log_warn "命令 $cmd 未找到，正在尝试安装..."
        install_packages "$pkg"
    else
        log_info "命令 $cmd 已就绪"
    fi
}

# 根据系统类型安装软件包
install_packages() {
    local packages=$1
    if [ -f /etc/fedora-release ]; then
        log_info "检测到 Fedora 系统，使用 dnf 安装: $packages"
        sudo dnf install -y $packages
    elif [ -f /etc/debian_version ] || [ -f /etc/lsb-release ]; then
        log_info "检测到 Ubuntu/Debian 系统，使用 apt 安装: $packages"
        sudo apt update && sudo apt install -y $packages
    else
        log_error "不支持的系统类型，无法自动安装 $packages"
    fi
}

# ==========================================
# 1. 前置检查
# ==========================================
[ "$EUID" -ne 0 ] && log_error "请使用 root 权限运行 (sudo su)"

# ==========================================
# 2. 系统检测与基础命令安装
# ==========================================
log_info "正在进行系统环境检测..."

# 检查并安装基础工具
check_and_install "wget" "wget"
check_and_install "tar" "tar"
check_and_install "make" "make"
check_and_install "gcc" "gcc"

# 安装编译依赖 (区分系统包名)
log_info "正在安装 Nginx 编译依赖..."
if [ -f /etc/fedora-release ]; then
    install_packages "pcre-devel zlib-devel openssl-devel"
elif [ -f /etc/debian_version ] || [ -f /etc/lsb-release ]; then
    install_packages "libpcre3-dev zlib1g-dev libssl-dev"
fi

# ==========================================
# 3. 下载并解压 Nginx
# ==========================================
log_info "正在下载 Nginx ${NGINX_VERSION}..."
mkdir -p ${TEMP_DIR} && cd ${TEMP_DIR}

if [ ! -f "nginx-${NGINX_VERSION}.tar.gz" ]; then
    wget -q "http://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz" || log_error "Nginx 源码下载失败"
else
    log_info "Nginx 源码包已存在，跳过下载"
fi

tar -zxf "nginx-${NGINX_VERSION}.tar.gz" && cd "nginx-${NGINX_VERSION}"

# ==========================================
# 4. 编译安装
# ==========================================
log_info "正在编译安装 Nginx (这可能需要几分钟)..."
./configure \
    --prefix=${INSTALL_PREFIX} \
    --pid-path=${PID_PATH} \
    --with-http_ssl_module \
    --with-http_v2_module \
    --with-http_stub_status_module \
    --with-http_gzip_static_module

make -j$(nproc) && make install

# ==========================================
# 5. 配置 Systemd 服务
# ==========================================
log_info "正在配置 Systemd 服务..."
cat > /etc/systemd/system/nginx.service << 'EOF'
[Unit]
Description=nginx - high performance web server
After=network.target

[Service]
Type=forking
PIDFile=/var/run/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s stop
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable nginx

# ==========================================
# 6. 生成优化版配置文件
# ==========================================
log_info "正在生成 nginx.conf 配置文件..."
cat > ${CONFIG_PATH} << 'EOF'
user  nginx;
worker_processes  auto;

error_log  logs/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  10240;
    use epoll;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  logs/access.log  main;

    sendfile        on;
    tcp_nopush      on;
    tcp_nodelay     on;
    keepalive_timeout  65;

    gzip  on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;

    server {
        listen       80;
        server_name  localhost;

        location / {
            root   html;
            index  index.html index.htm;
        }
    }
}
EOF

# 创建 nginx 用户
id -u nginx &>/dev/null || useradd -r -s /sbin/nologin nginx
chown -R nginx:nginx ${INSTALL_PREFIX}

# ==========================================
# 7. 防火墙配置 (兼容 firewalld 和 ufw)
# ==========================================
log_info "正在配置防火墙..."
if command -v firewall-cmd &> /dev/null; then
    log_info "检测到 firewalld，正在开放 80/443 端口..."
    firewall-cmd --permanent --add-service=http --add-service=https
    firewall-cmd --reload
elif command -v ufw &> /dev/null; then
    log_info "检测到 ufw，正在开放 80/443 端口..."
    ufw allow 80/tcp
    ufw allow 443/tcp
else
    log_warn "未检测到 firewalld 或 ufw，请手动开放 80/443 端口"
fi

# ==========================================
# 8. 启动验证
# ==========================================
log_info "正在启动 Nginx 服务..."
systemctl start nginx

if systemctl is-active --quiet nginx; then
    log_info "Nginx 部署成功！"
    log_info "配置文件: ${CONFIG_PATH}"
    log_info "访问地址: http://$(hostname -I | awk '{print $1}')"
else
    log_error "Nginx 启动失败，请使用 'systemctl status nginx' 查看详情"
fi

# 创建软连接，直接输入Nginx即可操作。
sudo ln -s /usr/local/nginx/sbin/nginx /usr/local/bin/nginx
```

### 脚本使用方法

1. **上传并授权**：

```bash
    chmod +x deploy_nginx.sh
```

2. **执行部署**：

```bash
    ./deploy_nginx.sh
```
    
3. **常用管理命令** ：

```bash
    # 启动/停止/重启
    systemctl start/stop/restart nginx
    
    # 重载配置 (不中断服务)
    nginx -s reload
    
    # 检查配置文件语法
    nginx -t
```

### 脚本亮点 (面试向)

1. **编译参数优化**：包含 `--with-http_ssl_module` (HTTPS)、`--with-http_stub_status_module` (状态监控) 等常用模块。
2. **Systemd 服务配置**：实现开机自启，标准的生产环境写法。
3. **配置文件模板**：内置 `worker_processes auto`、`epoll` 模型、Gzip 压缩等性能优化配置。
4. **安全加固**：使用 `nginx` 用户运行，避免 root 权限。