
cAdvisor（Container Advisor）是 Google 开源的容器资源监控工具，是 Docker/Kubernetes 生态容器监控的事实标准，Prometheus 官方推荐的容器级监控方案。

### 5.6.1 核心工作原理

cAdvisor 运行在宿主机上，通过 Linux 内核的 cgroup、namespace 机制，采集宿主机上所有容器的 CPU、内存、磁盘 IO、网络 IO、文件系统等资源使用指标，同时提供容器的运行状态、元数据信息，原生暴露 Prometheus 格式的`/metrics`接口，无需额外转换。

- Docker 环境下：需单独部署 cAdvisor 实例，采集宿主机上所有 Docker 容器的指标；
- Kubernetes 环境下：kubelet 已内置 cAdvisor，无需单独部署，直接通过 kubelet 的`/metrics/cadvisor`端点采集容器指标。

### 5.6.2 标准化部署

Docker 环境下的标准部署命令，需挂载宿主机的核心目录，保证 cAdvisor 能读取容器的 cgroup 数据与元数据：

```bash
docker run -d \
  --name=cadvisor \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --publish=8080:8080 \
  --restart=always \
  gcr.io/cadvisor/cadvisor:latest
```

- 默认监听端口：`8080`，默认指标路径：`/metrics`。

### 5.6.3 Prometheus 抓取配置

#### 1. Docker 环境 cAdvisor 抓取配置

```yaml
scrape_configs:
  - job_name: "cadvisor"
    scrape_interval: 15s
    static_configs:
      - targets: ["127.0.0.1:8080"]
        labels:
          env: prod
          host: "docker-node-01"
```

#### 2. Kubernetes 环境内置 cAdvisor 抓取配置

```yaml
scrape_configs:
  - job_name: "kubernetes-cadvisor"
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
```

### 5.6.4 核心指标与 PromQL 查询

cAdvisor 的核心指标覆盖容器全维度资源监控，关键指标与标准查询如下：

|监控维度|关键指标|标准 PromQL 查询|
|---|---|---|
|CPU 使用率|`container_cpu_usage_seconds_total`|容器 CPU 使用率：<br><br>`rate(container_cpu_usage_seconds_total{image!="", name!=""}[5m]) / on (id) group_left (name) container_spec_cpu_quota / 100000 * 100`|
|内存使用率|`container_memory_usage_bytes`、`container_spec_memory_limit_bytes`|容器内存使用率：<br><br>`container_memory_usage_bytes{image!="", name!=""} / container_spec_memory_limit_bytes{image!="", name!=""} * 100`|
|网络流量|`container_network_receive_bytes_total`、`container_network_transmit_bytes_total`|容器每秒网络流量：<br><br>`rate(container_network_receive_bytes_total[5m])`、`rate(container_network_transmit_bytes_total[5m])`|
|磁盘 IO|`container_fs_reads_bytes_total`、`container_fs_writes_bytes_total`|容器每秒磁盘 IO 吞吐量：<br><br>`rate(container_fs_reads_bytes_total[5m])`、`rate(container_fs_writes_bytes_total[5m])`|
|容器重启次数|`container_last_seen`、`container_start_time_seconds`|容器重启告警：<br><br>`changes(container_start_time_seconds{name!=""}[1h]) > 0`|

> 核心过滤说明：通过`image!=""`、`name!=""`过滤掉 pause 容器、系统容器、空闲 cgroup，仅保留业务容器的指标，降低时序基数。

### 5.6.5 生产环境最佳实践

1. **指标过滤**：通过`metric_relabel_configs`剔除无用的细粒度指标，仅保留核心资源指标，大幅降低时序基数；
2. **权限控制**：cAdvisor 仅需只读权限挂载宿主机目录，禁止授予写入权限，Docker 部署时所有挂载目录均使用`ro`只读模式；
3. **采集间隔**：容器监控推荐 15s 抓取间隔，保证指标精度的同时控制存储开销；
4. **Kubernetes 环境**：优先使用 kubelet 内置的 cAdvisor，禁止重复部署独立 cAdvisor 实例，避免指标重复与资源浪费；
5. **防火墙控制**：仅允许 Prometheus Server 访问 cAdvisor 的 8080 端口，禁止公网暴露。

---

## 本章小结

本章完成了 Prometheus Exporters 生态与埋点监控的系统性学习，核心内容包括：

1. 明确了 Exporters 在 Prometheus 监控体系中的核心定位与设计原则，建立了全场景监控的分类思路；
2. 掌握了 Node Exporter 的主机监控原理、部署配置与核心指标查询，实现了基础设施的标准化监控；
3. 深入理解了 Blackbox Exporter 的被动触发式探测机制，掌握了 ICMP/TCP/HTTP/DNS 四大探测模块的配置与典型应用场景；
4. 熟练掌握了 MySQL、Redis、Kafka 三大主流中间件 / 数据库的 Exporter 部署、权限配置与核心监控维度，建立了中间件监控的通用规范；
5. 建立了业务自定义埋点的标准化设计规范，完成了 Golang 与 Java/Spring Boot 两大生态的埋点实战，实现了业务白盒监控；
6. 理解了 cAdvisor 的容器监控原理，掌握了 Docker 与 Kubernetes 环境下的容器指标采集、过滤与查询方法；
7. 形成了 Exporter 全场景的生产级最佳实践，规避了安全风险、性能损耗与高基数问题。

本章内容是 Prometheus 监控体系的落地核心，覆盖了从基础设施、中间件、容器到业务应用的全链路监控能力，为下一章动态服务发现与 Relabeling 机制的学习建立了采集目标的基础。