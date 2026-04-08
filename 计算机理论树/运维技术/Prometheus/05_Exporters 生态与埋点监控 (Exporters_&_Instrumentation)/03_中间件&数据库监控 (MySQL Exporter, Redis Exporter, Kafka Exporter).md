
中间件与数据库是分布式系统的核心组件，Prometheus 生态提供了成熟的 Exporter 方案，覆盖绝大多数主流组件，本节聚焦生产环境最常用的 MySQL、Redis、Kafka 三大核心组件的监控实现。

### 5.4.1 通用设计原则

所有中间件 / 数据库 Exporter 均遵循以下通用原则：

1. **只读权限**：Exporter 仅通过中间件 / 数据库的原生管理接口、只读账号获取状态数据，无写入权限，避免对业务系统造成影响；
2. **无侵入性**：无需修改中间件 / 数据库的核心配置与业务代码，仅需创建只读账号、开启管理接口即可；
3. **标准化指标**：输出的指标遵循行业通用命名规范，覆盖性能、吞吐量、连接数、复制状态、异常等核心维度。

### 5.4.2 MySQL 监控：mysqld_exporter

`mysqld_exporter`是 Prometheus 官方维护的 MySQL/MariaDB 监控专用 Exporter，通过 MySQL 的`information_schema`、`performance_schema`与`SHOW STATUS`命令获取数据库状态指标，覆盖 MySQL 全维度监控。

#### 1. 前置准备：创建 MySQL 只读账号

```sql
-- 创建监控专用只读账号
CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'your_secure_password' WITH MAX_USER_CONNECTIONS 3;
-- 授予必要的权限
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
-- 刷新权限
FLUSH PRIVILEGES;
```

- 核心约束：`MAX_USER_CONNECTIONS`限制 Exporter 的最大连接数，避免监控连接耗尽数据库连接池。

#### 2. 标准化部署

- 二进制部署：从官方 GitHub 仓库获取对应架构的二进制包，创建配置文件`.my.cnf`存储数据库认证信息，启动服务；

    ```bash
    # .my.cnf 配置文件
    [client]
    user = exporter
    password = your_secure_password
    host = 127.0.0.1
    port = 3306
    ```
    
    ```bash
    ./mysqld_exporter --config.my-cnf=.my.cnf
    ```
    
- Docker 部署：
    
    ```bash
    docker run -d \
      --name mysqld_exporter \
      -p 9104:9104 \
      -e DATA_SOURCE_NAME="exporter:your_secure_password@(127.0.0.1:3306)/" \
      prom/mysqld-exporter:latest
    ```
    
- 默认监听端口：`9104`。

#### 3. Prometheus 抓取配置

```yaml
scrape_configs:
  - job_name: "mysql_exporter"
    scrape_interval: 15s
    static_configs:
      - targets: ["127.0.0.1:9104"]
        labels:
          env: prod
          mysql_cluster: "prod-mysql-01"
```

#### 4. 核心监控维度与关键指标

|监控维度|关键指标|核心 PromQL 查询|
|---|---|---|
|实例存活|`mysql_up`|实例存活状态：`mysql_up == 1`|
|连接数监控|`mysql_connections_max`、`mysql_threads_connected`|连接池使用率：`mysql_threads_connected / mysql_connections_max * 100`|
|查询吞吐量|`mysql_com_select_total`、`mysql_com_insert_total`、`mysql_com_update_total`、`mysql_com_delete_total`|每秒查询次数：`rate(mysql_com_select_total[5m])`|
|慢查询监控|`mysql_slow_queries_total`|慢查询每秒增长速率：`rate(mysql_slow_queries_total[5m])`|
|InnoDB 性能|`mysql_innodb_buffer_pool_hit_rate`、`mysql_innodb_row_ops_total`|缓冲池命中率：`mysql_innodb_buffer_pool_read_requests / (mysql_innodb_buffer_pool_read_requests + mysql_innodb_buffer_pool_reads) * 100`|
|主从复制|`mysql_slave_running`、`mysql_slave_sql_running`、`mysql_slave_lag_seconds`|从库复制中断告警：`mysql_slave_running == 0`；复制延迟告警：`mysql_slave_lag_seconds > 30`|

### 5.4.3 Redis 监控：redis_exporter

生产环境最主流的 Redis 监控 Exporter 为社区维护的`oliver006/redis_exporter`，是 Prometheus 生态适配最完善的 Redis 监控方案，兼容 Redis 2.x 至 7.x 全版本，支持单机、主从、哨兵、集群模式。

#### 1. 标准化部署

- Docker 部署（推荐）：

    ```bash
    docker run -d \
      --name redis_exporter \
      -p 9121:9121 \
      -e REDIS_ADDR="redis://127.0.0.1:6379" \
      -e REDIS_PASSWORD="your_redis_password" \
      oliver006/redis_exporter:latest
    ```
    
- 二进制部署：从 GitHub 仓库获取对应架构的二进制包，通过启动参数指定 Redis 地址与密码，默认监听端口`9121`。

#### 2. Prometheus 抓取配置

```yaml
scrape_configs:
  - job_name: "redis_exporter"
    scrape_interval: 15s
    static_configs:
      - targets: ["127.0.0.1:9121"]
        labels:
          env: prod
          redis_cluster: "prod-redis-cluster"
```

#### 3. 核心监控维度与关键指标

|监控维度|关键指标|核心 PromQL 查询|
|---|---|---|
|实例存活|`redis_up`|实例存活状态：`redis_up == 1`|
|性能监控|`redis_commands_processed_total`、`redis_command_call_duration_seconds_sum`|每秒命令执行次数：`rate(redis_commands_processed_total[5m])`|
|内存监控|`redis_memory_used_bytes`、`redis_memory_max_bytes`|内存使用率：`redis_memory_used_bytes / redis_memory_max_bytes * 100`|
|连接数监控|`redis_connected_clients`、`redis_maxclients`|客户端连接使用率：`redis_connected_clients / redis_maxclients * 100`|
|持久化监控|`redis_rdb_last_save_timestamp_seconds`、`redis_aof_enabled`|RDB 持久化失败告警：`redis_rdb_bgsave_in_progress == 0 and time() - redis_rdb_last_save_timestamp_seconds > 3600 * 24`|
|键空间监控|`redis_keyspace_keys_total`、`redis_keyspace_expired_keys_total`|过期键每秒增长：`rate(redis_keyspace_expired_keys_total[5m])`|
|复制监控|`redis_master_link_up`、`redis_replica_offset`|主从复制中断告警：`redis_master_link_up == 0`|

### 5.4.4 Kafka 监控：kafka_exporter

生产环境主流的 Kafka 监控 Exporter 为社区维护的`danielqsj/kafka_exporter`，通过 Kafka 的 Admin API 与 JMX 接口获取 Broker、Topic、消费者组的全维度指标，兼容 Kafka 0.10 + 全版本。

#### 1. 标准化部署

- Docker 部署：

    ```bash
    docker run -d \
      --name kafka_exporter \
      -p 9308:9308 \
      danielqsj/kafka_exporter:latest \
      --kafka.server=127.0.0.1:9092 \
      --kafka.server=127.0.0.2:9092
    ```
    
- 二进制部署：从 GitHub 仓库获取对应架构的二进制包，通过启动参数指定 Kafka Broker 地址列表，默认监听端口`9308`。

#### 2. Prometheus 抓取配置

```yaml
scrape_configs:
  - job_name: "kafka_exporter"
    scrape_interval: 30s
    static_configs:
      - targets: ["127.0.0.1:9308"]
        labels:
          env: prod
          kafka_cluster: "prod-kafka-cluster"
```

#### 3. 核心监控维度与关键指标

|监控维度|关键指标|核心 PromQL 查询|
|---|---|---|
|Broker 存活|`kafka_brokers`|在线 Broker 数量：`kafka_brokers`|
|消息吞吐量|`kafka_topic_messages_in_total`、`kafka_topic_bytes_in_total`|Topic 每秒消息入队量：`rate(kafka_topic_messages_in_total[5m])`|
|消费者组滞后量（Lag）|`kafka_consumergroup_lag`|消费者组滞后量 TOP10：`topk(10, sum by (consumergroup, topic) (kafka_consumergroup_lag))`|
|分区监控|`kafka_topic_partitions`、`kafka_topic_under_replicated_partitions`|未同步副本分区告警：`sum by (topic) (kafka_topic_under_replicated_partitions) > 0`|
|离线分区|`kafka_topic_offline_partitions`|离线分区告警：`sum by (topic) (kafka_topic_offline_partitions) > 0`|

### 5.4.5 生产环境最佳实践

1. **权限最小化**：为 Exporter 创建专用只读账号，仅授予监控所需的最小权限，禁止授予写入、修改、删库等高危权限；
2. **本地部署**：Exporter 优先与对应的中间件 / 数据库部署在同一台机器上，避免跨网络采集导致的指标延迟与抖动；
3. **采集间隔优化**：数据库 / 中间件监控推荐 15s-30s 抓取间隔，避免高频采集对组件造成性能压力；
4. **指标过滤**：通过`metric_relabel_configs`剔除无用的细粒度指标，降低时序基数，例如剔除单表、单分区的非核心指标；
5. **高可用部署**：核心组件的 Exporter 采用多实例部署，避免单点故障导致监控中断。

---
