
Metricbeat 是 Beats 家族的第二大核心组件，专门用于 **系统、服务、中间件的指标数据采集** ，是 Elastic Stack 可观测性平台的 Metrics 层核心采集工具，可替代传统的 Zabbix Agent、Node Exporter、Prometheus Exporter 等指标采集工具，与 Elasticsearch、Kibana 深度原生集成，开箱即用。

### 6.3.1 Metricbeat 核心架构与设计理念

Metricbeat 的核心设计理念是 **模块化、开箱即用** ，基于 Go 语言开发，具备与 Filebeat 一致的轻量级、低资源占用、高可靠性的特性，核心架构分为两大核心模块：

1. **Module（模块）**
    
    Module 是 Metricbeat 针对特定监控对象设计的采集套件，每个 Module 对应一个监控目标，例如`system`模块对应主机系统指标，`nginx`模块对应 Nginx 服务指标，`mysql`模块对应 MySQL 数据库指标。每个 Module 内置了默认的采集配置、指标解析规则、ES 索引模板、Kibana 可视化仪表盘，无需额外开发，启用后即可实现指标采集与可视化。
    
    Metricbeat 官方提供了上百个内置 Module，覆盖操作系统、数据库、中间件、消息队列、容器、云服务等全场景监控对象，是其核心优势。
    
1. **Metricset（指标集）**
    
    Metricset 是 Module 的子单元，每个 Metricset 对应 Module 中的一类具体指标，实现指标采集的精细化管理。例如`system`模块包含`cpu`、`memory`、`network`、`diskio`、`process`等多个 Metricset，用户可根据需求启用需要的 Metricset，无需采集全量指标，减少资源开销与存储成本。

Metricbeat 的完整工作流程：启用的 Module 通过 Metricset 定期从监控目标拉取指标数据，经过 Processor 预处理后，通过 Output 转发到 Elasticsearch，Kibana 通过内置的仪表盘直接展示指标数据，实现监控数据的采集、存储、可视化全链路闭环。

### 6.3.2 系统指标采集

系统指标采集是 Metricbeat 最基础、最常用的功能，通过内置的`system`模块实现，用于采集主机服务器的 CPU、内存、磁盘、网络、进程、系统负载等核心操作系统指标，是服务器监控的核心基础。

#### 标准生产级配置示例

```yaml
metricbeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false

# 【核心】system模块配置
metricbeat.modules:
  - module: system
    # 采集周期，默认10s，生产环境建议10s~60s
    period: 10s
    # 启用的指标集
    metricsets:
      - cpu        # CPU使用率、负载、上下文切换等指标
      - memory     # 内存使用率、可用内存、缓存、交换分区等指标
      - network    # 网络带宽、数据包、错误包、丢包率等指标
      - diskio     # 磁盘IO读写速率、IOPS、等待时间等指标
      - filesystem # 磁盘使用率、总容量、可用容量、inode使用率等指标
      - load       # 系统1分钟、5分钟、15分钟平均负载
      - process    # 进程级CPU、内存、IO指标
      - socket     # 网络连接状态指标
    # 进程指标过滤，仅采集指定进程，避免全量进程采集导致的性能开销
    process.include_top_n:
      by_cpu: 20  # 采集CPU占用前20的进程
      by_memory: 20 # 采集内存占用前20的进程
    # 文件系统过滤，排除不需要采集的挂载点
    filesystem.ignore_types: [nfs, smbfs, tmpfs, sysfs, proc, overlay]
    # 自定义字段，与Filebeat一致
    fields:
      env: prod
      region: guangzhou
    fields_under_root: true

# 输出到Elasticsearch
output.elasticsearch:
  hosts: ["http://192.168.1.10:9200"]
  username: elastic
  password: "your-es-password"
  index: "metricbeat-system-%{+yyyy.MM.dd}"

# Kibana仪表盘自动加载配置
setup.kibana:
  host: "http://192.168.1.10:5601"
setup.dashboards.enabled: true
```

#### 核心指标集说明

|Metricset|核心监控指标|核心应用场景|
|---|---|---|
|`cpu`|用户态 CPU 使用率、内核态 CPU 使用率、空闲 CPU、iowait、上下文切换次数、中断次数|服务器 CPU 瓶颈排查，高负载问题定位|
|`memory`|总内存、可用内存、已用内存、缓存内存、交换分区使用率、内存使用率|内存泄漏排查，服务器内存资源监控|
|`filesystem`|磁盘总容量、可用容量、已用容量、使用率、inode 总数、inode 使用率|磁盘空间不足告警，inode 耗尽问题排查|
|`diskio`|读写吞吐量、IOPS、平均 IO 等待时间、磁盘利用率|磁盘 IO 瓶颈排查，慢 IO 问题定位|
|`network`|入站 / 出站带宽、数据包数量、错误包数量、丢包率、TCP 连接状态|网络瓶颈排查，网络异常问题定位|
|`load`|1 分钟、5 分钟、15 分钟系统平均负载，CPU 核数归一化负载|系统整体负载监控，过载问题预警|
|`process`|进程 CPU / 内存 / IO 占用、进程状态、启动时间、文件句柄数|异常进程定位，业务服务资源占用监控|

#### 核心最佳实践

1. **采集周期优化**：常规服务器监控，采集周期建议设置为 30s~60s，无需高频采集，减少指标数据量与 ES 存储开销；核心业务服务器可设置为 10s；
2. **指标集精简**：仅启用需要的 Metricset，避免采集无用指标，例如无需监控进程细节时，禁用`process`指标集；
3. **进程过滤**：启用`process`指标集时，必须通过`process.include_top_n`或`process.include_names`过滤进程，严禁采集服务器全量进程指标，否则会导致 MetricbeatCPU 占用过高，指标数据量爆炸；
4. **文件系统过滤**：必须通过`filesystem.ignore_types`排除虚拟文件系统、网络文件系统，避免重复采集、无效指标采集；
5. **仪表盘自动加载**：启用`setup.dashboards.enabled: true`，Metricbeat 会自动在 Kibana 中创建对应的系统监控仪表盘，开箱即用，无需手动开发可视化图表。

### 6.3.3 中间件 / 服务指标采集

Metricbeat 通过内置的专用 Module，实现对主流中间件、数据库、消息队列等服务的指标采集，无需额外部署 Exporter，与 Elastic Stack 原生集成，是生产环境中间件监控的首选方案。

#### 核心采集流程与配置规范

1. **前置准备**：开启目标服务的状态监控接口，暴露指标数据。例如 Nginx 需启用`stub_status`模块，MySQL 需创建监控用户并授予权限，Redis 需开启状态查询接口；
2. **启用对应 Module**：Metricbeat 的 Module 默认存放在`modules.d/`目录下，通过`mv moduleName.yml.disabled moduleName.yml`启用对应 Module；
3. **配置 Module 参数**：修改 Module 配置文件，填写目标服务的地址、认证信息、采集周期、启用的 Metricset 等参数；
4. **启动 Metricbeat**：自动加载 Module 配置，采集指标并输出到 ES，同时自动加载 Kibana 内置仪表盘。

#### 常用中间件 Module 标准配置示例

##### 1. Nginx 指标采集

**前置准备**：Nginx 配置文件中启用`stub_status`模块：

```nginx
server {
    listen 80;
    server_name localhost;
    location /nginx_status {
        stub_status;
        allow 127.0.0.1;
        allow 192.168.1.0/24;
        deny all;
    }
}
```

**Metricbeat Nginx Module 配置（modules.d/nginx.yml）**：

```yaml
- module: nginx
  metricsets:
    - stubstatus
    - access
    - error
  period: 10s
  hosts: ["http://127.0.0.1:80"]
  # Nginx状态页地址
  server_status_path: "/nginx_status"
  # 日志采集路径
  access_logs:
    - path: "/var/log/nginx/access.log"
  error_logs:
    - path: "/var/log/nginx/error.log"
```

##### 2. MySQL 指标采集

**前置准备**：MySQL 中创建监控用户并授权：

```sql
CREATE USER 'metricbeat'@'%' IDENTIFIED BY 'your-password';
GRANT REPLICATION CLIENT, PROCESS, SELECT ON *.* TO 'metricbeat'@'%';
FLUSH PRIVILEGES;
```

**Metricbeat MySQL Module 配置（modules.d/mysql.yml）**：

```yaml
- module: mysql
  metricsets:
    - status
    - innodb
    - performance
    - schema
  period: 10s
  hosts: ["tcp(127.0.0.1:3306)/"]
  username: metricbeat
  password: "your-password"
```

##### 3. Redis 指标采集

**Metricbeat Redis Module 配置（modules.d/redis.yml）**：

```yaml
- module: redis
  metricsets:
    - info
    - keyspace
  period: 10s
  hosts: ["tcp://127.0.0.1:6379"]
  # Redis密码，无密码则注释
  password: "your-redis-password"
  # 监控的数据库编号
  db: 0
```

#### 核心最佳实践

1. **权限最小化**：为 Metricbeat 创建专用的监控用户，仅授予监控必需的最小权限，严禁使用管理员账号，避免安全风险；
2. **网络隔离**：中间件的监控接口仅对 Metricbeat 所在的服务器开放，严禁对公网暴露，避免未授权访问；
3. **采集周期匹配**：根据中间件的重要程度调整采集周期，核心数据库可设置为 10s，非核心中间件可设置为 60s；
4. **指标精简**：仅启用需要的 Metricset，例如无需监控表结构时，禁用 MySQL 的`schema`指标集，减少无效指标采集；
5. **原生仪表盘复用**：所有官方内置 Module 均提供了对应的 Kibana 仪表盘，启用后自动加载，无需手动开发，大幅降低监控平台搭建成本。

## 6.4 Beats 家族生产级核心避坑指南

1. **版本一致性强制规范**：Beats（Filebeat/Metricbeat）的大版本必须与 Elasticsearch、Logstash 的大版本完全一致，小版本尽量保持一致，避免出现 API 不兼容、数据格式异常、字段丢失等问题；
2. **Registry 文件保护**：Registry 文件是 Beats 断点续传的核心，生产环境必须对 Registry 文件所在的`data`目录做持久化存储，严禁删除该目录，否则会导致日志全量重复采集，ES 集群被打满；容器化部署时，必须将`data`目录挂载到宿主机持久化卷；
3. **文件句柄泄漏防控**：必须配置合理的文件句柄关闭策略，避免日志轮转后文件句柄未释放，导致磁盘空间无法释放；同时需调整服务器的最大文件句柄数，默认的 1024 完全无法满足生产环境采集需求，建议调整到 65535 以上；
4. **资源限制**：生产环境部署 Beats 时，必须通过 systemd、Docker 等方式设置 CPU、内存的资源上限，避免 Beats 异常时抢占业务服务的资源，影响业务稳定性；
5. **日志轮转适配**：必须适配业务日志的轮转策略，`ignore_older`必须大于日志轮转周期，`close.on_state_change`配置必须匹配日志写入频率，避免日志轮转后采集中断、重复采集；
6. **多行合并正则优化**：多行合并的正则表达式必须简洁高效，严禁使用复杂的回溯正则，避免 Beats CPU 占用率飙升；同时必须在测试环境充分验证，避免正则规则错误导致的日志合并错乱、数据丢失；
7. **安全加固**：生产环境必须启用 SSL/TLS 加密传输，Beats 与 ES/Logstash/Kafka 之间的通信必须加密；所有认证信息严禁明文写在配置文件中，应通过环境变量、密钥管理系统注入；
8. **监控与自愈**：必须对 Beats 自身的运行状态、采集延迟、转发成功率、资源占用做监控，配置异常告警；同时通过 systemd 配置进程自动重启，实现异常自愈。

## 本章小结

本章全面覆盖了 Beats 家族的核心知识点，核心掌握内容如下：

1. 深入理解了 Beats 家族的诞生背景与架构价值，明确了传统 ELK 采集层的核心痛点，掌握了 Beats 与 Logstash 的场景边界与生产级架构规范；
2. 精通了 Filebeat 的核心架构与工作原理，掌握了 inputs 日志源配置的核心规则与生产级优化方案；
3. 完全掌握了 multiline 多行合并的核心规则，精通了 Java 异常堆栈等场景的标准配置，解决了跨多行日志的合并难题；
4. 掌握了 fields 自定义字段的配置方法与标准化规范，理解了元数据标签对后续检索、聚合的核心价值；
5. 精通了 Filebeat 三大核心输出类型的配置方法、适用场景与架构选型规范，可根据业务规模选择合适的输出链路；
6. 理解了 Metricbeat 的模块化架构，掌握了系统指标采集的配置与核心指标集，具备了主流中间件的指标采集与监控能力；
7. 掌握了 Beats 生产环境部署的核心避坑指南与最佳实践，具备了大规模边缘节点采集的标准化部署与运维能力。