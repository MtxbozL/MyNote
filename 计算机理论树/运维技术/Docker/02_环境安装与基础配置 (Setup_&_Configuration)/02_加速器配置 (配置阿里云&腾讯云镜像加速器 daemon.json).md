
Docker 默认镜像源为 Docker Hub，服务器位于海外，国内网络环境下存在访问延迟高、镜像拉取超时、拉取失败等问题。镜像加速器是国内云厂商提供的 Docker Hub 镜像缓存站点，可大幅提升镜像拉取速度与可用性，是国内环境必须配置的核心项。

### 2.1 配置核心文件：daemon.json

Docker 引擎的所有全局配置均通过`daemon.json`文件实现，该文件是 Docker Daemon 的核心配置文件，优先级高于启动命令行参数。

- Linux 系统配置文件路径：`/etc/docker/daemon.json`（文件不存在时需手动创建，严禁直接修改默认配置文件）
- Docker Desktop 配置入口：图形化界面 → Settings → Docker Engine，直接在编辑框内修改 JSON 配置

### 2.2 加速器配置规范

1. **配置格式**
    
    在`daemon.json`中通过`registry-mirrors`字段配置加速器地址，格式为 JSON 数组，示例如下：

    ```json
    {
      "registry-mirrors": [
        "https://<你的专属ID>.mirror.aliyuncs.com",
        "https://mirror.ccs.tencentyun.com",
        "https://docker.mirrors.ustc.edu.cn"
      ]
    }
    ```
    
2. **加速器地址说明**
    
    - 阿里云加速器：需登录阿里云控制台，进入「容器镜像服务」→「镜像工具」→「镜像加速器」获取个人专属加速器地址，为国内稳定性最高的加速器；
    - 腾讯云加速器：腾讯云内网用户可直接使用，公网用户有带宽限制；
    - 中科大镜像源：教育网公开镜像源，无专属地址要求，适合个人开发环境使用。
    
3. **配置生效步骤（Linux 环境）**

    ```bash
    # 重新加载系统服务配置
    sudo systemctl daemon-reload
    # 重启Docker Daemon使配置生效
    sudo systemctl restart docker
    ```
    
    Docker Desktop 环境修改配置后，点击「Apply & Restart」即可自动生效。
    

### 2.3 配置验证与注意事项

1. **生效验证**：配置生效后，可通过后续章节的`docker info`命令验证加速器是否成功加载；
2. **语法规范**：`daemon.json`必须严格遵循 JSON 语法规范，字段间使用英文逗号分隔，最后一个字段后严禁添加逗号，否则会导致 Docker Daemon 启动失败；
3. **配置合并**：若`daemon.json`已存在其他配置项，需将`registry-mirrors`字段合并到现有 JSON 结构中，严禁直接覆盖文件导致原有配置丢失。
