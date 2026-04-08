# 获取与推送：连接云端仓库的纽带 (search, pull, login, push)

在 Docker 的生态中，**Registry（注册服务器，如 Docker Hub）** 扮演着应用商店的角色。我们将按照一个完整的“检索 -> 下载 -> 鉴权 -> 上传”的工程逻辑，剖析这四个高频命令。

### 1. `docker search`：检索云端镜像

在获取镜像之前，我们通常需要先在仓库中定位它。虽然通过浏览器访问 Docker Hub 官网检索体验更好，但命令行提供了更极客的途径。

*   **基础语法**：`docker search [关键字]`
*   **示例**：`docker search nginx`
*   **核心输出字段解析**：
    *   **NAME**：镜像名称。没有前缀的（如 `nginx`）属于官方镜像；带有前缀的（如 `bitnami/nginx`）属于第三方组织或个人上传的镜像。
    *   **STARS**：点赞数，反映了该镜像的社区受欢迎程度。
    *   **OFFICIAL**：是否为官方维护（显示为 [OK]）。
*   **👨‍💻 学术建议 / 最佳实践**：在生产环境中，**强烈建议仅使用 OFFICIAL 标识的官方镜像**，或者知名开源组织（如 bitnami）维护的镜像。个人上传的镜像可能存在不可控的后门或安全漏洞。

### 2. `docker pull`：拉取镜像至本地

找到所需的镜像后，需要将其从远程 Registry 下载到本地宿主机。

*   **基础语法**：`docker pull [Registry域名]/[命名空间]/[仓库名]:[标签Tag]`
*   **示例与隐式路由分析**：
    当你输入 `docker pull nginx` 时，Docker 引擎实际上执行的是：
    `docker pull docker.io/library/nginx:latest`
    *   `docker.io`：默认的官方 Registry 域名。
    *   `library`：官方镜像的专属命名空间。
    *   `latest`：默认的标签。如果不指定 Tag，Docker 会自动寻找名为 `latest` 的最新版本。
*   **底层机制剖析（UnionFS 下载特性）**：
    在拉取过程中，你会看到多行类似 `Pull complete` 的输出。这就是我们在 1.4 节讲过的 **分层联合文件系统 (UnionFS)** 的具象体现。Docker 会**并发地下载多个只读层**。更巧妙的是，如果某个基础层（比如底层 Ubuntu 环境）你本地的其它镜像已经下载过了，Docker 会通过计算 SHA256 哈希值发现它，并直接显示 `Already exists`（已存在），跳过下载。这就是 Docker 极度节省带宽的原因。

### 3. `docker login`：身份鉴权

当你只做“下载公开镜像”的操作时，无需登录。但如果你想下载企业的**私有镜像**，或者将自己构建的镜像**推送**到云端，就必须进行身份认证。

*   **基础语法**：`docker login [Registry域名]` (不加域名则默认登录 Docker Hub)
*   **交互过程**：输入你的 Username 和 Password。
*   **安全机制**：登录成功后，认证凭证默认会以 Base64 编码的形式保存在宿主机的 `~/.docker/config.json` 文件中。后续的操作将自动读取此凭证。
*   **登出命令**：`docker logout`。

### 4. `docker push`：推送镜像至云端

开发完成后，我们将本地打包好的镜像推送到远端仓库，供运维拉取部署。**这是 CI/CD 流水线中最核心的一环。**

*   **避坑指南（前置条件）**：
    很多初学者会直接执行 `docker push my-app:1.0`，结果遇到 `denied: requested access to the resource is denied`（拒绝访问）错误。
    **原因**：你不能把镜像推送到官方的 `library` 命名空间下。你必须先使用 `docker tag` 命令，将镜像重命名为**你自己的账号命名空间**。
*   **正确推送流转步骤**：
    1.  **打标签 (Tag)**：`docker tag my-app:1.0 username/my-app:1.0`
        *(将 username 替换为你在 Docker Hub 注册的用户名)*
    2.  **执行推送**：`docker push username/my-app:1.0`
*   **底层机制剖析**：
    与 `pull` 命令类似，推送时 Docker 同样基于分层检查。如果远程仓库已经存在某些基础层，引擎只会将**本地独有且新增的增量层（如你的代码层）**上传，极大地提升了分发效率。