
Blackbox Exporter 是 Prometheus 官方维护的**黑盒探测专用 Exporter**，核心实现了端到端的黑盒监控能力，无需在目标机器上部署任何代理，即可完成网络可达性、服务可用性、端口存活、SSL 证书状态等探测，是白盒监控的核心补充。

### 5.3.1 核心工作机制

Blackbox Exporter 的工作模式与常规 Exporter 存在本质区别，采用**被动触发式探测**机制：

1. Prometheus Server 向 Blackbox Exporter 的`/probe`接口发起抓取请求，通过 URL 参数传递**探测目标、探测模块、超时时间**等配置；
2. Blackbox Exporter 接收请求后，按照指定的模块与参数，对目标执行对应的探测操作；
3. 探测完成后，Blackbox Exporter 将探测结果转换为 Prometheus 标准化指标，返回给 Prometheus Server；
4. Prometheus Server 对返回的指标进行存储、计算与告警。

该机制的核心优势是：**一套 Blackbox Exporter 实例可支持任意数量目标的探测**，无需为每个目标部署独立的 Exporter，适配大规模网络与服务探测场景。

### 5.3.2 标准化部署与核心配置

#### 1. 部署方式

- **二进制部署**：从官方 GitHub 仓库获取对应架构的二进制包，解压后直接启动，默认监听端口`9115`，默认配置文件`blackbox.yml`；
- **Docker 快速部署**：
    
    ```bash
    docker run -d \
      --name blackbox_exporter \
      -p 9115:9115 \
      -v /path/to/blackbox.yml:/etc/blackbox_exporter/blackbox.yml \
      prom/blackbox-exporter:latest
    ```

#### 2. 核心配置文件`blackbox.yml`

配置文件采用 YAML 格式，核心定义**探测模块（modules）**，每个模块指定探测类型、超时时间、探测参数等，官方默认提供四大核心探测模块的标准配置：

```yaml
modules:
  # ICMP ping探测模块：用于主机网络可达性探测
  icmp:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: "ip4" # 优先使用IPv4
  # TCP端口探测模块：用于端口存活、TCP服务可用性探测
  tcp:
    prober: tcp
    timeout: 5s
    tcp:
      preferred_ip_protocol: "ip4"
  # HTTP接口探测模块：用于HTTP/HTTPS接口可用性、状态码、SSL证书探测
  http:
    prober: http
    timeout: 10s
    http:
      method: GET
      preferred_ip_protocol: "ip4"
      valid_http_versions: ["HTTP/1.1", "HTTP/2"]
      valid_status_codes: [200, 204, 301, 302] # 合法的HTTP状态码
      fail_if_ssl: false
      fail_if_not_ssl: false
      tls_config:
        insecure_skip_verify: false # 禁用SSL证书校验
  # DNS域名解析探测模块：用于DNS服务可用性、域名解析正确性探测
  dns:
    prober: dns
    timeout: 5s
    dns:
      preferred_ip_protocol: "ip4"
      query_name: "prometheus.io"
      query_type: "A"
      valid_rcodes: ["NOERROR"]
```

### 5.3.3 Prometheus 抓取配置

Blackbox Exporter 的抓取配置核心是通过`params`传递探测模块，通过`relabel_configs`动态生成探测目标参数，是配置的关键。

#### 1. ICMP ping 探测配置示例

```yaml
scrape_configs:
  - job_name: "blackbox_icmp"
    metrics_path: /probe
    params:
      module: [icmp] # 指定使用icmp探测模块
    static_configs:
      - targets:
          - 192.168.1.10
          - 192.168.1.11
          - www.baidu.com
        labels:
          env: prod
    relabel_configs:
      # 将targets中的目标地址设置为probe请求的target参数
      - source_labels: [__address__]
        target_label: __param_target
      # 将探测目标地址设置为instance标签，便于查询
      - source_labels: [__param_target]
        target_label: instance
      # 将抓取地址替换为Blackbox Exporter的地址
      - target_label: __address__
        replacement: 127.0.0.1:9115 # Blackbox Exporter的实际访问地址
```

#### 2. HTTP 接口探测配置示例

```yaml
scrape_configs:
  - job_name: "blackbox_http"
    metrics_path: /probe
    params:
      module: [http] # 指定使用http探测模块
    static_configs:
      - targets:
          - https://prometheus.io
          - https://www.example.com/api/health
        labels:
          env: prod
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115
```

#### 3. TCP 端口探测配置示例

```yaml
scrape_configs:
  - job_name: "blackbox_tcp"
    metrics_path: /probe
    params:
      module: [tcp] # 指定使用tcp探测模块
    static_configs:
      - targets:
          - 192.168.1.10:3306 # MySQL端口
          - 192.168.1.10:6379 # Redis端口
        labels:
          env: prod
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115
```

### 5.3.4 核心指标与典型应用场景

Blackbox Exporter 探测返回的核心指标与典型应用场景如下：

1. **探测成功状态指标**：`probe_success`，值为 1 表示探测成功，0 表示探测失败，是服务存活告警的核心指标；
    
    - 告警示例：`probe_success{job="blackbox_http"} == 0`，触发接口不可用告警；
    
2. **探测耗时指标**：`probe_duration_seconds`，表示单次探测的总耗时，用于网络质量、服务响应性能监控；
3. **SSL 证书指标**：`probe_ssl_earliest_cert_expiry`，表示 SSL 证书的过期时间戳，用于证书过期预警；
    
    - 预警示例：`probe_ssl_earliest_cert_expiry - time() < 30 * 24 * 3600`，提前 30 天预警证书即将过期；
    
4. **DNS 解析指标**：`probe_dns_lookup_time_seconds`，表示 DNS 解析耗时，用于 DNS 服务性能监控；
5. **TCP 连接指标**：`probe_tcp_connect_time_seconds`，表示 TCP 三次握手耗时，用于网络链路质量监控。

### 5.3.5 生产环境最佳实践

1. **分布式部署**：跨机房、跨地域部署 Blackbox Exporter 实例，实现多地域的端到端探测，模拟不同地区用户的访问体验；
2. **超时时间配置**：探测超时时间必须小于 Prometheus 的`scrape_timeout`，避免抓取超时导致指标丢失；
3. **探测频率控制**：核心服务推荐 15s-30s 探测间隔，非核心服务推荐 1m-5m 探测间隔，避免探测流量占用过多带宽；
4. **权限控制**：ICMP 探测需要 CAP_NET_RAW 权限，Linux 环境下需为二进制文件授予对应权限，Docker 环境需添加`--cap-add=NET_RAW`参数；
5. **探测目标隔离**：不同类型的探测目标分 Job 配置，便于标签管理、告警规则配置与权限控制。

---
