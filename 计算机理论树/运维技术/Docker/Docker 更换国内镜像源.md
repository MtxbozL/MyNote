## 一、目的

解决官方镜像源拉取慢、超时问题，大幅提升镜像下载速度

## 二、常用国内镜像源（选 2-3 个搭配即可）

- 阿里云：需登录「阿里云容器服务」获取专属加速地址（推荐，稳定高速）
- 腾讯云：`https://mirror.ccs.tencentyun.com`
- 网易云：`https://hub-mirror.c.163.com`
- 中科大：`https://docker.mirrors.ustc.edu.cn`

## 三、操作步骤

### （一）Linux 系统（通用：Ubuntu/CentOS/Debian 等）

1. 检查 Docker 状态（确保已安装并运行）

```bash
docker version
# 或
systemctl status docker
```

2. 创建 / 编辑 Docker 配置文件

```bash
sudo vi /etc/docker/daemon.json
```

（文件不存在则直接创建；**必须严格遵循 JSON 格式**，无 trailing comma）

3. 写入镜像源配置（示例，可替换为你的源）

json

```
{
  "registry-mirrors": [
    "https://mirror.ccs.tencentyun.com",
    "https://hub-mirror.c.163.com",
    "https://docker.mirrors.ustc.edu.cn"
  ]
}
```

4. 重载配置并重启 Docker 服务

bash

运行

```
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### （二）Windows / Mac Docker Desktop

1. 打开 Docker Desktop 界面
2. 进入 **Settings（设置）→ Docker Engine**
3. 在配置框中添加 `registry-mirrors` 字段（同 Linux JSON 格式）
4. 点击 **Apply & Restart** 重启 Docker

## 四、验证是否生效

执行以下命令，查看输出末尾的 `Registry Mirrors` 部分：

bash

运行

```
docker info
```

如果包含你配置的镜像源地址，说明更换成功

## 五、常见问题

1. **JSON 格式错误**：`daemon.json` 必须用双引号、无多余逗号，否则 Docker 重启失败
2. **权限不足**：编辑配置文件时加 `sudo`，或切换 root 用户
3. **重启失败排查**：执行 `journalctl -u docker.service` 查看具体错误日志