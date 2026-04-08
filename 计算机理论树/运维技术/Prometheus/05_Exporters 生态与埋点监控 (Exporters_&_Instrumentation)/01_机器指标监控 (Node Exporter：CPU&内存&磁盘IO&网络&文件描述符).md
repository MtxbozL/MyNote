## 本章学习目标

1. 明确 Exporters 在 Prometheus 监控体系中的核心定位与分类，建立全场景监控的覆盖思路
2. 掌握 Node Exporter 的核心原理、部署配置与主机全维度指标采集能力，实现基础设施标准化监控
3. 深入理解 Blackbox Exporter 的工作机制，掌握 ICMP/TCP/HTTP/DNS 四大黑盒探测模块的配置与应用
4. 熟练掌握主流中间件与数据库 Exporter 的部署、权限配置与核心指标监控，覆盖 MySQL、Redis、Kafka 三大核心组件
5. 建立业务自定义埋点的标准化规范，掌握 Golang 与 Java/Spring Boot 两大生态的埋点实战方法
6. 理解 cAdvisor 的容器监控原理，掌握 Docker 容器级资源指标的采集、过滤与查询方法
7. 形成 Exporter 选型、部署、配置的生产级最佳实践，规避安全风险、性能损耗与高基数问题

---

## 5.1 Exporters 核心定位与生态分类

Exporters 是 Prometheus 监控体系的**指标采集前端**，是连接 Prometheus Server 与被监控对象的核心桥梁。其核心职责是：将异构系统（主机、中间件、数据库、容器、业务应用）的原生状态数据，转换为符合 Prometheus 时序数据模型、支持 Pull 拉取的标准化指标格式，并通过 HTTP `/metrics` 接口暴露，供 Prometheus Server 采集。

### 5.1.1 核心设计原则

所有合规的 Exporters 均遵循以下核心设计原则：

1. **无状态化**：Exporter 仅负责指标的采集与格式转换，不做持久化存储、告警判断与聚合计算，所有计算逻辑均由 Prometheus Server 通过 PromQL 实现；
2. **无侵入性**：官方 / 社区 Exporters 无需修改被监控对象的代码，仅通过被监控系统的原生管理接口、状态文件实现数据采集；
3. **标准化格式**：输出的指标严格遵循 Prometheus 文本格式规范，明确指标名称、类型、标签、帮助文本与数值，保证语义一致性；
4. **轻量低耗**：Exporter 本身资源占用极低，避免对被监控对象造成额外的性能压力。

### 5.1.2 生态分类

Prometheus Exporters 生态覆盖全场景监控需求，可分为四大核心类别，对应本章的全部内容：

1. **基础设施类 Exporters**：面向主机、服务器、物理资源的监控，核心代表为**Node Exporter**；
2. **黑盒探测类 Exporters**：面向网络、服务可用性的端到端探测，核心代表为**Blackbox Exporter**；
3. **中间件 / 数据库类 Exporters**：面向各类中间件、数据库、消息队列的监控，覆盖 MySQL、Redis、Kafka、Nginx 等主流组件；
4. **容器监控类 Exporters**：面向 Docker/Kubernetes 容器资源监控，核心代表为**cAdvisor**；
5. **自定义业务埋点客户端库**：面向业务应用的自定义指标埋点，提供 Golang、Java、Python 等多语言官方客户端库，实现业务白盒监控。

---

## 5.2 机器指标监控：Node Exporter

Node Exporter 是 Prometheus 官方维护的**主机 / 服务器监控标准 Exporter**，采用 Go 语言开发，无任何第三方依赖，可跨平台运行，是基础设施监控的必备组件。

### 5.2.1 核心工作原理

Node Exporter 的核心采集逻辑是**直接读取操作系统内核暴露的状态文件**，无需特权进程，无需侵入系统：

- Linux 环境下：读取`/proc`、`/sys`伪文件系统，获取 CPU、内存、磁盘、网络、进程等内核状态数据；
- Windows/macOS 环境下：适配对应操作系统的原生系统调用，获取系统状态数据。

Node Exporter 采用**模块化设计**，每个采集能力对应一个独立的 Collector（采集器），可通过启动参数灵活启用 / 禁用，默认启用通用采集模块，禁用高危、高开销、平台专属模块。

### 5.2.2 标准化部署

#### 1. 二进制部署（生产环境首选）

```bash
# 1. 从官方下载地址获取对应架构的最新稳定版二进制包
wget https://github.com/prometheus/node_exporter/releases/download/v<version>/node_exporter-<version>.linux-amd64.tar.gz
# 2. 解压并安装
tar -zxvf node_exporter-<version>.linux-amd64.tar.gz
cd node_exporter-<version>.linux-amd64
cp node_exporter /usr/local/bin/
# 3. 配置systemd服务实现开机自启
cat > /etc/systemd/system/node_exporter.service << EOF
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=nobody
ExecStart=/usr/local/bin/node_exporter
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
# 4. 启动服务并设置开机自启
systemctl daemon-reload
systemctl start node_exporter
systemctl enable node_exporter
```

- 默认监听端口：`9100`，默认指标路径：`/metrics`；
- 可通过`--web.listen-address`参数修改监听端口，通过`--collector.<name>`启用指定采集模块，`--no-collector.<name>`禁用指定模块。

#### 2. Docker 快速部署

```bash
docker run -d \
  --name node_exporter \
  --net="host" \
  --pid="host" \
  -v "/:/host:ro,rslave" \
  quay.io/prometheus/node-exporter:latest \
  --path.rootfs=/host
```

- 核心参数说明：`--net="host"`使用宿主机网络，`--pid="host"`访问宿主机进程命名空间，`-v /:/host`挂载宿主机根目录，保证能读取`/proc`、`/sys`文件系统。

### 5.2.3 Prometheus 抓取配置

在 Prometheus 的`scrape_configs`中添加对应 Job，实现 Node Exporter 指标的采集：

```yaml
scrape_configs:
  - job_name: "node_exporter"
    scrape_interval: 15s
    scrape_timeout: 10s
    static_configs:
      - targets: ["192.168.1.10:9100", "192.168.1.11:9100"]
        labels:
          env: "prod"
          idc: "guangzhou-az1"
```

### 5.2.4 核心采集模块与关键指标

Node Exporter 默认启用的核心模块覆盖主机全维度监控，对应核心指标与 PromQL 查询如下：

|监控维度|核心采集模块|关键指标|标准 PromQL 查询|
|---|---|---|---|
|CPU 监控|`cpu`|`node_cpu_seconds_total`（CPU 各模式累计运行时间）|主机 CPU 使用率：<br><br>`100 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100`|
|内存监控|`meminfo`|`node_memory_MemTotal_bytes`（总内存）、`node_memory_MemAvailable_bytes`（可用内存）、`node_memory_SwapTotal_bytes`（Swap 总大小）|内存使用率：<br><br>`100 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100`|
|磁盘空间监控|`filesystem`|`node_filesystem_size_bytes`（分区总大小）、`node_filesystem_avail_bytes`（分区可用空间）|磁盘分区使用率：<br><br>`100 - (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100`|
|磁盘 IO 监控|`diskstats`|`node_disk_read_bytes_total`、`node_disk_written_bytes_total`、`node_disk_io_time_seconds_total`|磁盘每秒读写吞吐量：<br><br>`rate(node_disk_read_bytes_total[5m])`、`rate(node_disk_written_bytes_total[5m])`|
|网络监控|`netdev`|`node_network_receive_bytes_total`、`node_network_transmit_bytes_total`、`node_network_receive_drop_total`|网卡每秒流量：<br><br>`rate(node_network_receive_bytes_total[5m])`、`rate(node_network_transmit_bytes_total[5m])`|
|文件描述符监控|`filefd`|`node_filefd_allocated`、`node_filefd_maximum`|文件描述符使用率：<br><br>`node_filefd_allocated / node_filefd_maximum * 100`|
|系统负载监控|`loadavg`|`node_load1`、`node_load5`、`node_load15`|1 分钟系统平均负载：<br><br>`node_load1`|
|主机存活监控|内置|`up`|主机节点存活状态：<br><br>`up{job="node_exporter"} == 1`|

### 5.2.5 生产环境最佳实践

1. **模块裁剪**：禁用不需要的采集模块，减少资源占用与指标基数，例如禁用`arp`、`nf_conntrack`等非必要模块；
2. **权限最小化**：使用普通用户运行 Node Exporter，禁止使用 root 用户，仅授予必要的文件系统读取权限；
3. **指标过滤**：通过`metric_relabel_configs`剔除无用指标，降低 Prometheus 存储压力，下一章将详细讲解；
4. **采集间隔**：基础设施监控推荐 10s-15s 抓取间隔，保证指标精度的同时控制存储开销；
5. **防火墙控制**：仅允许 Prometheus Server 访问 9100 端口，禁止公网暴露。

---
