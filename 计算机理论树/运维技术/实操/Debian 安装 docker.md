### Debian Docker 安装脚本 (`install_docker_debian.sh`)

```bash
#!/bin/bash
set -e

# ==========================================
# 颜色输出
# ==========================================
COLOR_GREEN="\033[32m"
COLOR_RED="\033[31m"
COLOR_YELLOW="\033[33m"
COLOR_RESET="\033[0m"

log_info() { echo -e "${COLOR_GREEN}[INFO] $1${COLOR_RESET}"; }
log_warn() { echo -e "${COLOR_YELLOW}[WARN] $1${COLOR_RESET}"; }
log_error() { echo -e "${COLOR_RED}[ERROR] $1${COLOR_RESET}"; exit 1; }

# ==========================================
# 1. 前置检查
# ==========================================
[ "$EUID" -ne 0 ] && log_error "请使用 root 权限运行此脚本 (sudo su)"

log_info "正在检测系统环境..."
if [ ! -f /etc/debian_version ]; then
    log_error "此脚本仅适用于 Debian 系统"
fi

# ==========================================
# 2. 清理旧版本 (如果有)
# ==========================================
log_info "正在清理可能存在的旧版本 Docker..."
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do
    apt-get remove -y $pkg 2>/dev/null || true
done

# ==========================================
# 3. 安装基础依赖
# ==========================================
log_info "正在更新包索引并安装依赖..."
apt-get update
apt-get install -y ca-certificates curl gnupg lsb-release

# ==========================================
# 4. 添加 Docker 官方 GPG 密钥
# ==========================================
log_info "正在添加 Docker 官方 GPG 密钥..."
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

# ==========================================
# 5. 设置 Docker 稳定版仓库
# ==========================================
log_info "正在配置 Docker APT 仓库..."
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

# ==========================================
# 6. 安装 Docker Engine
# ==========================================
log_info "正在安装 Docker Engine (最新版)..."
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# ==========================================
# 7. 启动并设置开机自启
# ==========================================
log_info "正在启动 Docker 服务..."
systemctl start docker
systemctl enable docker

# ==========================================
# 8. 验证安装
# ==========================================
log_info "正在验证 Docker 安装..."
if docker run --rm hello-world; then
    log_info "Docker 安装成功！"
    log_info "Docker 版本: $(docker --version)"
    log_info "Docker Compose 版本: $(docker compose version)"
else
    log_error "Docker 验证失败，请检查服务状态"
fi
```

### 脚本使用方法

1. **创建并授权脚本**：
    
    bash
    
    运行
    
    ```
    # 创建脚本文件
    nano install_docker_debian.sh
    
    # 将上面的代码粘贴进去，保存退出 (Ctrl+O, Enter, Ctrl+X)
    
    # 赋予执行权限
    chmod +x install_docker_debian.sh
    ```
    
2. **执行安装**：
    
    bash
    
    运行
    
    ```
    ./install_docker_debian.sh
    ```
    
3. **后续常用命令**：
    
    bash
    
    运行
    
    ```
    # 查看 Docker 状态
    systemctl status docker
    
    # 运行一个测试容器
    docker run -d -p 80:80 docker/getting-started
    ```