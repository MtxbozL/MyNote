
Input 插件是 Logstash 管道的入口，负责上游数据的接入与 Event 的生成，是数据处理的第一环节。本节聚焦目录指定的三大核心 Input 插件，覆盖 99% 的生产级部署场景，明确其适用场景、标准配置与核心参数。

### 7.2.1 beats Input【核心对接 Beats 家族】

beats Input 是生产环境最常用的 Input 插件，专门用于对接 Beats 家族（Filebeat/Metricbeat 等）的输出，基于 Lumberjack 协议实现数据传输，内置 TLS 加密、压缩传输、负载均衡、批量接收等能力，是经典 ELK 架构的标准数据接入方案。

#### 核心适用场景

- 经典架构：`Filebeat → Logstash → Elasticsearch`，边缘节点 Filebeat 采集日志，集中式 Logstash 集群做数据清洗；
- 多 Beats 统一接入：Filebeat、Metricbeat、Packetbeat 等多类 Beats 数据统一接入 Logstash，做集中化处理。

#### 标准生产级配置

```ruby
input {
  beats {
    # 【必填】监听端口，默认5044，需与Filebeat的output.logstash端口一致
    port => 5044
    # 监听地址，0.0.0.0表示监听所有网卡
    host => "0.0.0.0"
    # 客户端连接数上限，避免连接数过多导致内存溢出
    client_inactivity_timeout => 3600
    # 批量接收大小，平衡吞吐量与内存开销
    batch_size => 2048
    # 接收缓冲区大小
    receive_buffer_bytes => 1048576
    # TLS加密配置，生产环境强制启用
    ssl_enabled => false
    ssl_certificate => "/etc/logstash/certs/server.crt"
    ssl_key => "/etc/logstash/certs/server.key"
    ssl_client_authentication => "none"
    # 编解码格式，默认json_lines，适配Beats的输出格式
    codec => "json_lines"
  }
}
```

#### 核心参数详解

- `port`：必填参数，监听的 TCP 端口，需确保服务器防火墙开放该端口，多管道部署时需使用不同端口；
- `host`：监听的 IP 地址，生产环境建议指定内网网卡 IP，而非 0.0.0.0，降低安全风险；
- `ssl_enabled`：生产环境必须启用 TLS 加密，避免数据明文传输，需与 Filebeat 的 SSL 配置双向匹配；
- `codec`：数据编解码器，默认`json_lines`适配 Beats 的 JSON 格式输出，严禁修改为 plain 格式，否则会导致数据解析异常。

### 7.2.2 kafka Input【企业级高可用架构核心】

kafka Input 是大规模生产环境的首选 Input 插件，用于对接 Kafka 消息队列，实现日志数据的削峰填谷、解耦采集与消费，是企业级高可用架构`Filebeat → Kafka → Logstash → Elasticsearch`的核心对接环节。

#### 核心适用场景

- 高并发大流量场景：大促、业务高峰期日志洪峰场景，通过 Kafka 实现流量削峰，避免下游 ES/Logstash 被打垮；
- 多消费者场景：同一份日志数据需要同时供给日志分析、安全审计、数据备份等多个业务方消费；
- 高可用保障：Logstash 集群滚动升级、故障重启时，数据暂存在 Kafka 中，无数据丢失风险。

#### 标准生产级配置

```ruby
input {
  kafka {
    # 【必填】Kafka集群地址，多个节点用逗号分隔
    bootstrap_servers => "192.168.1.30:9092,192.168.1.31:9092,192.168.1.32:9092"
    # 【必填】消费的Topic，支持通配符，如"filebeat-*"
    topics => ["filebeat-prod-java", "filebeat-prod-nginx"]
    # 【必填】消费者组ID，同一消费者组的实例分摊消费，保证消息不重复消费
    group_id => "logstash-prod-consumer"
    # 消费起始位置：earliest从头消费，latest从最新位置消费，默认latest
    auto_offset_reset => "latest"
    # 会话超时时间，超过该时间未发送心跳，被判定为下线，触发rebalance
    session_timeout_ms => 30000
    # 单次拉取最大记录数
    max_poll_records => 2048
    # 拉取超时时间
    fetch_max_wait_ms => 500
    # 自动提交offset间隔，生产环境建议手动提交，保证至少一次消费
    enable_auto_commit => true
    auto_commit_interval_ms => 5000
    # 安全认证配置，生产环境推荐SASL认证
    security_protocol => "PLAINTEXT"
    # 编解码格式，默认json
    codec => "json"
    # 消费者线程数，建议与Kafka分区数保持一致，最大化消费能力
    consumer_threads => 6
  }
}
```

#### 核心参数详解

- `bootstrap_servers`：Kafka 集群地址，建议配置至少 3 个节点，避免单节点故障导致消费中断；
- `topics`：支持单个 Topic、多个 Topic 数组、通配符 Topic，生产环境建议按业务、环境拆分 Topic，便于精细化管理；
- `group_id`：消费者组唯一标识，同一消费者组内的多个 Logstash 实例会分摊 Topic 的分区，实现水平扩展；
- `consumer_threads`：消费者线程数，最优值为 Topic 的分区总数，线程数超过分区数无任何性能提升，反而会增加线程调度开销；
- `auto_offset_reset`：生产环境建议设置为`latest`，避免重启后全量消费历史数据，导致 ES 集群压力激增；
- `enable_auto_commit`：高可靠场景建议设置为`false`，手动提交 offset，确保数据处理完成并成功输出后再提交 offset，避免数据丢失。

### 7.2.3 file Input

file Input 用于读取本地磁盘上的日志文件，实现与 Filebeat 类似的本地文件采集能力，基于文件尾追踪、断点续传机制实现日志的实时采集。

#### 核心适用场景

- 单机极简部署：Logstash 与业务服务部署在同一台服务器，无需额外部署 Filebeat，直接采集本地日志；
- 离线日志回溯：批量处理历史归档日志文件，一次性导入 ES 做数据分析。

#### 标准配置示例

```ruby
input {
  file {
    # 【必填】日志文件路径，支持通配符
    path => ["/var/log/nginx/*.log", "/opt/app/logs/*.log"]
    # 排除不需要采集的文件
    exclude => ["*.gz", "*.tmp", "debug.log"]
    # 采集起始位置：beginning从头采集，end从文件末尾采集，默认end
    start_position => "end"
    # .sincedb文件路径，记录文件读取偏移量，实现断点续传
    sincedb_path => "/var/lib/logstash/sincedb"
    # 文件扫描频率，默认15s
    stat_interval => 10s
    # 文件句柄关闭策略
    close_older => 3600
    # 忽略超过指定时间未修改的文件
    ignore_older => 72h
    # 多行合并配置，与Filebeat的multiline逻辑一致
    codec => multiline {
      pattern => "^%{TIMESTAMP_ISO8601}"
      negate => true
      what => "previous"
    }
  }
}
```

#### 与 Filebeat 的核心选型边界

|对比维度|Logstash file Input|Filebeat|
|---|---|---|
|资源占用|极高，JVM 依赖，内存占用 500MB+|极低，无 JVM 依赖，内存占用 10~50MB|
|核心能力|采集 + 清洗一体化|仅采集，无清洗能力|
|部署位置|不建议部署在业务服务器|推荐部署在业务服务器边缘节点|
|适用场景|单机极简部署、离线日志回溯|生产环境大规模边缘采集|

**生产级规范**：严禁在业务服务器部署 Logstash 使用 file Input 采集日志，必须使用 Filebeat 做边缘采集，Logstash 集中部署做清洗，实现架构解耦。
