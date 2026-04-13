
Output 插件是 Logstash 管道的出口，负责将清洗完成的标准化 Event 路由输出到下游目标，完成数据的持久化与转发。本节聚焦目录指定的两大核心 Output 插件，覆盖生产环境的核心输出场景，明确其动态路由、性能优化、高可用配置方案。

### 7.4.1 elasticsearch Output【核心输出目标】

elasticsearch Output 是生产环境最核心的 Output 插件，负责将处理完成的 Event 批量写入 Elasticsearch 集群，与 ES 深度原生集成，支持动态索引路由、索引模板管理、ILM 生命周期管理、负载均衡、重试机制等企业级能力。

#### 7.4.1.1 核心配置与集群对接

```ruby
output {
  elasticsearch {
    # 【必填】ES集群地址，多个节点用逗号分隔，实现负载均衡
    hosts => ["http://192.168.1.10:9200", "http://192.168.1.11:9200", "http://192.168.1.12:9200"]
    # ES认证信息
    user => "elastic"
    password => "your-es-password"
    # 【核心】目标索引名称，支持变量引用，实现动态索引路由
    index => "logstash-%{[service]}-%{+yyyy.MM.dd}"
    # 索引生命周期管理配置
    ilm_enabled => true
    ilm_policy => "logstash-ilm-policy"
    ilm_rollover_alias => "logstash-%{[service]}"
    # 批量提交配置，性能优化核心
    bulk_size => 2048
    flush_size => 2048
    idle_flush_time => 5
    # 重试机制
    retry_max_interval => 30
    retry_on_conflict => 5
    # 负载均衡与健康检查
    sniffing => true
    sniffing_interval => 30
    # 压缩传输
    http_compression => true
    # 文档ID配置，仅需幂等写入时指定
    # document_id => "%{[uuid]}"
  }
}
```

#### 7.4.1.2 动态索引路由实现

动态索引路由是 elasticsearch Output 的核心能力，通过变量引用实现按业务维度、时间维度自动拆分索引，是日志数据管理的标准规范。核心变量包括：

1. **字段变量**：通过`%{[字段名]}`引用 Event 中的字段值，例如`%{[service]}`、`%{[env]}`，实现按服务、环境拆分索引；
2. **时间变量**：通过`%{+yyyy.MM.dd}`引用`@timestamp`字段的时间格式，实现按天 / 月拆分索引，是日志场景的标准用法。

##### 生产级动态索引示例

1. **按服务 + 日期拆分索引**：不同服务的日志写入不同的索引，便于独立管理、权限控制、性能优化
    
    ```ruby
    index => "log-%{[env]}-%{[service]}-%{+yyyy.MM.dd}"
    ```
    
    最终生成的索引名示例：`log-prod-user-service-2026.04.10`
    
2. **按日志级别分流索引**：ERROR 级别的错误日志单独写入专用索引，便于问题排查与告警监控
    
    ```ruby
    output {
      if [level] == "error" {
        elasticsearch {
          hosts => ["http://192.168.1.10:9200"]
          index => "log-error-%{[service]}-%{+yyyy.MM.dd}"
          user => "elastic"
          password => "your-es-password"
        }
      } else {
        elasticsearch {
          hosts => ["http://192.168.1.10:9200"]
          index => "log-normal-%{[service]}-%{+yyyy.MM.dd}"
          user => "elastic"
          password => "your-es-password"
        }
      }
    }
    ```
    

#### 7.4.1.3 批量提交与性能优化

elasticsearch Output 的写入性能核心由批量提交参数决定，生产环境必须根据集群规模、数据流量优化配置：

- `bulk_size`：单次批量请求的文档数量，默认 1000，生产环境建议设置为 1000~5000，根据单文档大小调整，单文档越大，该值应越小；
- `flush_size`：内存中缓存的文档数量达到该值时，触发批量提交，建议与`bulk_size`保持一致；
- `idle_flush_time`：空闲超时时间，即使缓存的文档数量未达到`flush_size`，超过该时间也会触发提交，默认 5s，避免数据延迟；
- `http_compression`：启用 HTTP 压缩，减少网络传输带宽开销，生产环境必须开启；
- `sniffing`：开启集群节点自动发现，自动感知 ES 集群的节点变化，实现负载均衡，生产环境建议开启。

#### 7.4.1.4 索引生命周期管理（ILM）衔接

elasticsearch Output 原生支持 ES 的索引生命周期管理（ILM），可直接关联 ILM 策略，实现索引的自动滚动、冷热迁移、过期删除，无需额外配置，是生产环境降低存储成本的核心方案。核心配置：

```ruby
output {
  elasticsearch {
    hosts => ["http://192.168.1.10:9200"]
    ilm_enabled => true
    # 关联ES中提前创建的ILM策略
    ilm_policy => "logstash-7d-hot-30d-warm-delete"
    # 滚动别名，必须与ILM策略中的别名一致
    ilm_rollover_alias => "log-prod-service"
    # 索引模式
    ilm_pattern => "{now/d}-000001"
  }
}
```

### 7.4.2 file Output（数据备份与归档）

file Output 用于将处理后的 Event 写入本地磁盘文件，核心用于日志数据的备份、归档、离线存储，实现数据的多副本冗余，避免 ES 集群故障导致的数据丢失。

#### 标准生产级配置

```ruby
output {
  file {
    # 【必填】输出文件路径，支持变量
    path => "/data/logstash/backup/%{[service]}/%{+yyyy.MM.dd}.log"
    # 写入格式，默认json，生产环境推荐json_lines，每行一个JSON对象
    codec => json_lines
    # 刷新间隔，默认2s
    flush_interval => 5
    # 文件轮转策略
    file_mode => 0644
    filename_failure => "failed-%{+yyyy.MM.dd}.log"
    # gzip压缩配置，归档场景推荐开启
    gzip => false
  }
}
```

#### 核心适用场景

1. **数据备份**：核心业务日志同时写入 ES 和本地文件，实现双副本备份，满足合规审计要求；
2. **离线归档**：历史日志写入归档文件，压缩后存储到对象存储，长期留存；
3. **异常数据分流**：Grok 匹配失败、解析异常的数据，单独写入本地文件，便于后续回溯排查。

### 7.4.3 多输出目标与条件路由

Logstash 支持同时配置多个 Output 插件，结合条件判断实现数据的多链路输出、动态路由，是企业级架构的常用方案。

#### 典型生产示例：多目标同时输出

```ruby
output {
  # 主链路：写入ES集群
  elasticsearch {
    hosts => ["http://192.168.1.10:9200"]
    index => "log-%{[env]}-%{[service]}-%{+yyyy.MM.dd}"
    user => "elastic"
    password => "your-es-password"
  }

  # 备份链路：写入Kafka，供其他业务方消费
  if [env] == "prod" {
    kafka {
      bootstrap_servers => "192.168.1.30:9092"
      topic_id => "log-prod-backup"
      codec => json
    }
  }

  # 异常数据分流：写入本地文件
  if "_grok_failure" in [tags] or "_date_failure" in [tags] {
    file {
      path => "/data/logstash/failed/%{+yyyy.MM.dd}.log"
      codec => json_lines
    }
  }

  # 错误日志告警：输出到标准输出，配合告警系统监控
  if [level] == "error" {
    stdout {
      codec => json_lines
    }
  }
}
```
