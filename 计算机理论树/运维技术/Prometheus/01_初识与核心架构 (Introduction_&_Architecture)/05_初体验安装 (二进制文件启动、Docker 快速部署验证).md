
本节提供 Prometheus 的两种标准化部署方式，分别为二进制文件部署（适用于生产环境）与 Docker 快速部署（适用于学习验证场景），完成部署后可实现 Prometheus 的基础功能验证。

### 1.5.1 环境前置要求

- 操作系统：Linux/Windows/macOS，生产环境推荐使用 Linux 发行版（CentOS 7+/Ubuntu 20.04+）；
- 架构：amd64/arm64，Prometheus 提供多架构的官方二进制包与 Docker 镜像；
- 网络：服务器需开放 9090 端口（Prometheus Server 默认 Web 端口），确保与监控目标的网络可达。

### 1.5.2 二进制文件部署

二进制文件部署是生产环境的推荐方案，无额外依赖，可直接部署运行，步骤如下：

1. **获取官方二进制包**
    
    从 Prometheus 官方下载地址（[https://prometheus.io/download/](https://prometheus.io/download/)）获取对应操作系统与架构的最新稳定版二进制包，执行解压命令：

    ```bash
    tar -zxvf prometheus-<version>.linux-<arch>.tar.gz
    cd prometheus-<version>.linux-<arch>
    ```
    
2. **核心文件说明**
    
    解压后核心文件包括：
    
    - `prometheus`：Prometheus Server 主程序二进制文件；
    - `promtool`：Prometheus 配置文件、规则文件的校验工具；
    - `prometheus.yml`：Prometheus 核心配置文件；
    - `console_libraries/`、`consoles/`：Web 控制台的模板文件。
    
3. **启动运行**
    
    执行以下命令启动 Prometheus Server，默认加载当前目录下的 prometheus.yml 配置文件，默认监听 9090 端口：

    ```bash
    ./prometheus --config.file=prometheus.yml
    ```
    
4. **基础配置校验**
    
    通过 promtool 工具校验配置文件的合法性，命令如下：

    ```bash
    ./promtool check config prometheus.yml
    ```
    

### 1.5.3 Docker 快速部署验证

Docker 部署方式适用于快速学习与功能验证，无需配置环境依赖，一键启动，步骤如下：

1. **拉取官方 Docker 镜像**
    
    执行以下命令拉取 Prometheus 官方最新稳定版镜像：

    ```bash
    docker pull prom/prometheus:latest
    ```
    
2. **启动容器实例**
    
    执行以下命令启动 Prometheus 容器，映射 9090 端口，挂载本地配置文件至容器内：

    ```bash
    docker run -d \
      -p 9090:9090 \
      -v /path/to/local/prometheus.yml:/etc/prometheus/prometheus.yml \
      --name prometheus \
      prom/prometheus:latest
    ```
    
3. **容器运行状态校验**
    
    执行`docker ps`命令查看容器运行状态，确认 Prometheus 容器正常启动。

### 1.5.4 部署结果验证

1. **Web UI 访问验证**
    
    部署完成后，通过浏览器访问`http://<服务器IP>:9090`，可正常打开 Prometheus 原生 Web 控制台，说明服务启动成功。
2. **指标采集验证**
    
    Prometheus 默认开启自身指标的采集，在 Web 控制台的 Graph 页面，输入 PromQL 语句`prometheus_http_requests_total`，执行查询可获取 Prometheus 自身的 HTTP 请求指标数据，说明指标抓取、存储与查询能力正常。

---

## 本章小结

本章完成了 Prometheus 监控体系的入门基础学习，核心内容包括：

1. 建立了监控体系的核心理论框架，明确了白盒 / 黑盒监控的分类，掌握了四个黄金指标、USE 原则、RED 方法三大标准化监控方法论的适用场景与核心逻辑；
2. 明确了 Prometheus 的项目定位、生态地位与核心设计理念，理解了其作为云原生监控事实标准的核心优势；
3. 掌握了 Prometheus 四大核心架构组件的职责与协同逻辑，建立了完整的架构认知；
4. 区分了 Pull 与 Push 两种监控模型的核心差异、适用边界与选型原则，明确了 Prometheus 主 Pull 辅 Push 的设计逻辑；
5. 完成了 Prometheus 两种标准化部署方式的实操，实现了基础环境的功能验证。

本章为后续深入学习 Prometheus 的配置管理、数据模型、PromQL 查询、告警机制等核心内容建立了完整的知识基础与运行环境。