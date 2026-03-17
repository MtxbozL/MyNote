### 1. 拉取 Linux 镜像（以 Ubuntu 为例）

先从 Docker Hub 拉取一个常用的 Linux 发行版镜像（如 Ubuntu、CentOS 等）：

```bash
docker pull ubuntu:latest
```

### 2. 启动并进入容器

使用 `run` 命令启动容器，并通过 `-it` 参数进入交互模式：

```bash
docker run -it --name my-shell ubuntu /bin/bash
```

- `-it`：保持交互模式并分配伪终端（关键，否则容器会立即退出）。
- `--name my-shell`：给容器起个名字（方便后续管理）。
- `ubuntu`：使用的镜像名。
- `/bin/bash`：启动后执行的 Shell（也可以用 `/bin/sh`）。

如果希望创建并后台运行 Ubuntu（exit 不退出）：

```bash
docker run -d --name myubuntu ubuntu sleep infinity
```

- `-d`：后台运行（守护进程）
- `--name myubuntu`：给容器起个名字
- `sleep infinity`：让容器**永远不退出**

进入容器：

```bash
docker exec -it myubuntu /bin/bash
```

### 3. 退出与重新进入容器

- **退出容器**：在容器内输入 `exit` 或按 `Ctrl+D`，容器会停止。
- **重新启动已停止的容器**：

```bash
    docker start my-shell
```

- **再次进入运行中的容器**：

```bash
    docker exec -it my-shell /bin/bash
```

### 4. 常用容器管理命令

- 查看所有容器（包括停止的）：`docker ps -a`
- 停止容器：`docker stop my-shell`
- 删除容器：`docker rm my-shell`