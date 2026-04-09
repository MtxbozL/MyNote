
企业级生产环境中，基础的 stub_status 模块无法满足长期时序存储、多维度可视化、异常告警、多实例集群监控的需求，**Prometheus + Nginx Exporter + Grafana**是行业标准的开源监控解决方案，可实现 Nginx 集群的全维度、可视化、可预警的全栈监控体系。

### 5.1 监控架构与核心组件角色

该体系分为三大核心组件，各司其职，形成完整的监控闭环：

|组件名称|核心角色与功能|
|---|---|
|**Nginx Exporter**|部署在 Nginx 服务器上，负责抓取 Nginx stub_status 模块的指标，转换为 Prometheus 兼容的时序数据格式，暴露`/metrics`接口供 Prometheus 抓取|
|**Prometheus**|时序数据库，按照配置的抓取规则，定期从 Nginx Exporter 拉取监控指标，完成数据存储、时序计算，提供 PromQL 查询语言支持指标查询与聚合|
|**Grafana**|可视化与告警平台，对接 Prometheus 数据源，通过拖拽式面板实现监控指标的可视化展示，配置异常告警规则，支持邮件、钉钉、企业微信等多渠道告警通知|

### 5.2 部署与配置全流程

#### 5.2.1 前置条件

Nginx 服务器已启用`stub_status`模块，配置了可访问的状态接口（如`/nginx_status`），且 Nginx Exporter 服务器可访问该接口。

#### 5.2.2 Nginx Exporter 部署

Nginx Exporter 由 Prometheus 社区官方维护，推荐使用二进制部署或 Docker 部署两种方式。

##### 1. 二进制部署（生产环境推荐）

```bash
# 1. 下载最新版本（替换为最新release版本号）
wget https://github.com/nginxinc/nginx-prometheus-exporter/releases/download/v1.3.0/nginx-prometheus-exporter_1.3.0_linux_amd64.tar.gz
# 2. 解压安装
tar -zxvf nginx-prometheus-exporter_1.3.0_linux_amd64.tar.gz
mv nginx-prometheus-exporter /usr/local/bin/
# 3. 验证安装
nginx-prometheus-exporter --version
```

##### 2. 配置 Systemd 服务，实现开机自启

创建服务文件`/etc/systemd/system/nginx-exporter.service`：

```bash
[Unit]
Description=Nginx Prometheus Exporter
After=network.target nginx.service

[Service]
Type=simple
User=nginx
# 核心启动参数：指定Nginx状态接口地址，监听端口默认9113
ExecStart=/usr/local/bin/nginx-prometheus-exporter \
    --nginx.scrape-uri=https://status.example.com/nginx_status \
    --web.listen-address=0.0.0.0:9113 \
    --nginx.ssl-verify=false
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

启动服务并设置开机自启：

```bash
systemctl daemon-reload
systemctl start nginx-exporter
systemctl enable nginx-exporter
# 验证服务状态
systemctl status nginx-exporter
# 验证metrics接口
curl http://127.0.0.1:9113/metrics
```

##### 3. Docker 部署

```bash
docker run -d \
  --name nginx-exporter \
  -p 9113:9113 \
  nginx/nginx-prometheus-exporter:latest \
  --nginx.scrape-uri=https://status.example.com/nginx_status
```

#### 5.2.3 Prometheus 配置

修改 Prometheus 配置文件`prometheus.yml`，添加 Nginx 监控抓取任务：

```yaml
global:
  scrape_interval: 15s # 全局抓取间隔15秒
  evaluation_interval: 15s

scrape_configs:
  # Nginx集群监控任务
  - job_name: "nginx_cluster"
    static_configs:
      # 所有Nginx实例的Exporter地址
      - targets:
        - "192.168.1.10:9113"
        - "192.168.1.11:9113"
        - "192.168.1.12:9113"
        labels:
          env: "production"
          project: "web_service"
```

配置完成后，重启 Prometheus，在 Prometheus Web UI 的`Targets`页面中，可看到 Nginx 实例的状态为`UP`，说明抓取正常。

#### 5.2.4 Grafana 可视化配置

1. **添加 Prometheus 数据源**：登录 Grafana，进入「Configuration」→「Data Sources」→「Add data source」，选择 Prometheus，填写 Prometheus 服务地址，保存并测试连接。
2. **导入官方监控看板**：Grafana 官方市场提供了成熟的 Nginx 监控看板，推荐使用看板 ID：**12708**（Nginx Exporter 官方看板），进入「Dashboards」→「Import」，输入看板 ID，选择 Prometheus 数据源，点击导入即可。
3. **核心看板指标**：导入完成后，可看到完整的监控面板，核心指标包括：
    
    - 服务存活状态、实例在线数
    - 实时 QPS、请求量趋势
    - 活跃连接数、新建连接数趋势
    - 请求状态码分布、4xx/5xx 错误率
    - TCP 连接复用率、长连接空闲数
    - 请求读写状态分布、慢请求占比
    - 多实例集群维度的聚合统计
    

### 5.3 核心告警规则配置

生产环境必须配置核心告警规则，实现异常事件的及时通知，告警规则配置在 Prometheus 的`rules`文件中，核心告警规则示例如下：

```yaml
groups:
- name: nginx_alerts
  rules:
  # 告警1：Nginx实例宕机，Exporter无法抓取
  - alert: NginxInstanceDown
    expr: up{job="nginx_cluster"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Nginx实例宕机"
      description: "实例 {{ $labels.instance }} 已超过1分钟无法抓取，服务可能宕机"

  # 告警2：5xx错误率过高，超过1%
  - alert: Nginx5xxErrorRateHigh
    expr: sum(rate(nginx_http_responses_total{code=~"5.."}[5m])) / sum(rate(nginx_http_responses_total[5m])) * 100 > 1
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "Nginx 5xx错误率过高"
      description: "实例 {{ $labels.instance }} 5xx错误率已达 {{ $value | printf \"%.2f\" }}%，超过阈值1%"

  # 告警3：QPS突降，超过50%
  - alert: NginxQpsDrop
    expr: (sum(rate(nginx_http_requests_total[5m])) / sum(rate(nginx_http_requests_total[5m] offset 10m)) - 1) * 100 < -50
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Nginx QPS突降"
      description: "集群QPS较10分钟前下降 {{ $value | printf \"%.2f\" }}%，业务可能异常"

  # 告警4：活跃连接数超过阈值
  - alert: NginxActiveConnectionsHigh
    expr: nginx_connections_active > 10000
    for: 3m
    labels:
      severity: warning
    annotations:
      summary: "Nginx活跃连接数过高"
      description: "实例 {{ $labels.instance }} 活跃连接数已达 {{ $value }}，超过阈值10000"
```

### 5.4 进阶监控扩展

1. **日志指标监控**：通过`filebeat`采集 Nginx 访问日志，提取请求耗时、状态码、接口维度的指标，输出到 Prometheus，实现接口级别的精细化监控，定位慢接口、异常接口。
2. **SSL 证书监控**：通过`blackbox_exporter`监控 HTTPS 证书有效期，提前 30 天触发证书过期告警，避免证书过期导致服务不可用。
3. **集群高可用监控**：针对 Nginx+Keepalived 高可用集群，监控 VIP 漂移状态、Keepalived 进程存活、主备节点状态，实现集群层面的高可用监控。
4. **多租户权限控制**：Grafana 中配置多租户权限，不同业务线的运维人员仅可查看对应 Nginx 实例的监控看板，满足企业级权限管控要求。

### 5.5 最佳实践与注意事项

1. **Exporter 安全防护**：Nginx Exporter 的 9113 端口必须通过防火墙限制访问，仅允许 Prometheus 服务器 IP 访问，禁止公网开放，避免监控指标泄露。
2. **抓取间隔规范**：生产环境抓取间隔建议设置为 15s，高频交易场景可调整为 10s，避免抓取过于频繁导致服务器性能损耗，同时保证异常事件的及时捕获。
3. **告警分级规范**：告警必须分级，`critical`级别（服务宕机、5xx 错误率飙升）仅发送给核心运维人员，`warning`级别仅工作时段通知，避免告警风暴。
4. **数据持久化**：Prometheus 建议配置远程存储（如 Thanos、M3DB），实现监控数据的长期存储，满足合规审计要求；同时配置 Prometheus 高可用集群，避免监控单点故障。
5. **全链路监控整合**：将 Nginx 监控与后端服务、数据库、服务器主机监控整合到同一 Grafana 看板，实现从接入层到应用层的全链路可观测，故障定位效率提升 10 倍以上。