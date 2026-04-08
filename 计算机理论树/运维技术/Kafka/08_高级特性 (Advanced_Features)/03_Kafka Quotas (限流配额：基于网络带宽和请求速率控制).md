
Kafka Quotas（配额）是 Kafka 原生的**流量管控与资源隔离机制**，核心解决多租户场景下，单个客户端过度占用集群的网络、CPU、内存资源，导致其他客户端被 “饿死”、集群稳定性下降的问题。通过配额机制，可精细化管控每个客户端的资源使用上限，实现集群资源的公平分配与多租户隔离，是企业级 Kafka 集群的必备管控能力。

### 8.3.1 配额的核心类型与管控维度

Kafka 配额分为两大核心类型，分别管控集群最核心的两类资源：网络带宽资源与 CPU 计算资源，覆盖了客户端对集群资源占用的核心场景。

#### 1. 网络带宽配额

网络带宽配额管控客户端的网络传输速率，以字节 / 秒为单位，分为生产带宽配额与消费带宽配额两类，是最常用的配额类型：

- **生产带宽配额（producer_byte_rate）**：限制单个客户端每秒向 Broker 发送的消息字节总数，管控生产端的入流量；
- **消费带宽配额（consumer_byte_rate）**：限制单个客户端每秒从 Broker 拉取的消息字节总数，管控消费端的出流量。

#### 2. 请求处理速率配额

请求处理速率配额管控客户端请求对 Broker CPU 资源的占用率，以 Broker 请求处理线程的时间占比为单位，核心参数为`request_percentage`：

- 该参数定义了客户端请求可占用的 Broker 请求处理线程总时间的最大百分比，取值范围 0-100；
- 例如设置为 10，代表该客户端的所有请求，累计占用的请求处理时间不得超过 Broker 总处理时间的 10%，避免单个客户端的大量请求耗尽 Broker 的 CPU 资源。

### 8.3.2 配额的生效维度与优先级

Kafka 配额支持三个精细化的生效维度，可实现从集群全局到单个客户端的分级管控，同时支持默认配额与精准配额的组合配置，适配多租户集群的复杂管控需求。

#### 1. 生效维度

- **用户维度（User）**：基于客户端的认证用户 Principal 生效，适用于开启了身份认证的多租户集群，为不同的业务用户设置不同的配额；
- **客户端 ID 维度（Client ID）**：基于生产者 / 消费者客户端配置的`client.id`生效，无需开启身份认证即可使用，适用于按业务系统、应用实例管控配额；
- **用户 + 客户端 ID 维度**：基于「用户 + 客户端 ID」的组合生效，是最精细化的管控维度，可实现同一个用户下不同客户端的差异化配额管控。

#### 2. 优先级规则

配额的匹配遵循**最精准匹配优先**的原则，优先级从高到低为：

`用户+客户端ID配额 > 用户维度配额 > 客户端ID维度配额 > 用户维度默认配额 > 客户端ID维度默认配额 > 集群全局默认配额`

若客户端未匹配到任何精准配额，则使用对应维度的默认配额；若未配置默认配额，则默认无配额限制。

### 8.3.3 配额的实现原理：软限流延迟惩罚机制

与绝大多数中间件的硬限流（超过阈值直接拒绝请求）不同，Kafka 采用**软限流延迟惩罚机制**实现配额管控，核心逻辑为：

1. Broker 会实时统计每个客户端的资源使用情况，当客户端的资源使用量超过配额阈值时，不会直接拒绝客户端的请求；
2. 而是计算出该客户端需要延迟的时间，将请求的响应延迟该时间后再返回给客户端，通过降低客户端的请求频率，将其资源使用量限制在配额阈值以内；
3. 延迟时间的计算基于漏桶算法，保证客户端的资源使用速率长期稳定在配额阈值附近，无突发流量。

该设计的核心优势在于：避免了硬限流导致的客户端频繁重试，反而加重集群负载的问题，同时保证了客户端的请求不会丢失，仅会增加轻微的延迟，完美适配 Kafka 高吞吐、高可靠的设计目标。

### 8.3.4 配额配置方式与实战命令

Kafka 配额支持**动态配置**与**静态配置**两种方式，生产环境强烈推荐使用动态配置，无需重启 Broker 即可实时生效，通过`kafka-configs.sh`工具完成配置。

#### 1. 核心配置命令示例

所有命令均基于 KRaft 架构，使用`--bootstrap-server`指定集群地址，兼容 ZK 架构。

##### 示例 1：为客户端 ID 设置带宽配额

```bash
# 为客户端ID order_service_producer 设置生产带宽配额10MB/s，消费带宽配额20MB/s
./bin/kafka-configs.sh --bootstrap-server localhost:9092 --alter \
  --entity-type clients --entity-name order_service_producer \
  --add-config producer_byte_rate=10485760,consumer_byte_rate=20971520
```

##### 示例 2：为用户设置请求处理速率配额

```bash
# 为认证用户 finance_user 设置请求处理时间占比上限10%
./bin/kafka-configs.sh --bootstrap-server localhost:9092 --alter \
  --entity-type users --entity-name finance_user \
  --add-config request_percentage=10
```

##### 示例 3：为用户 + 客户端 ID 设置精细化配额

```bash
# 为用户 finance_user 下的客户端ID pay_consumer 设置消费带宽配额5MB/s
./bin/kafka-configs.sh --bootstrap-server localhost:9092 --alter \
  --entity-type users --entity-name finance_user \
  --entity-type clients --entity-name pay_consumer \
  --add-config consumer_byte_rate=5242880
```

##### 示例 4：设置客户端 ID 维度的默认配额

```bash
# 为所有未配置精准配额的客户端，设置默认生产带宽配额5MB/s，消费带宽配额10MB/s
./bin/kafka-configs.sh --bootstrap-server localhost:9092 --alter \
  --entity-type clients --entity-default \
  --add-config producer_byte_rate=5242880,consumer_byte_rate=10485760
```

##### 示例 5：查看与删除配额

```bash
# 查看指定客户端ID的配额配置
./bin/kafka-configs.sh --bootstrap-server localhost:9092 --describe \
  --entity-type clients --entity-name order_service_producer

# 删除指定客户端的生产带宽配额
./bin/kafka-configs.sh --bootstrap-server localhost:9092 --alter \
  --entity-type clients --entity-name order_service_producer \
  --delete-config producer_byte_rate
```

### 8.3.5 生产环境最佳实践

1. **默认配额兜底原则**：多租户集群必须设置集群默认配额，避免未配置精准配额的客户端过度占用集群资源，保障集群的基础稳定性；
2. **分级配额管控**：核心业务系统设置较高的配额，非核心业务、测试业务设置较低的配额，实现资源的优先级分配，保障核心业务的稳定性；
3. **配额阈值预留缓冲**：配额阈值建议设置为业务峰值流量的 1.5~2 倍，避免业务正常的流量波动触发限流，影响业务性能；
4. **配额监控与告警**：监控客户端的配额使用率、限流延迟时间，当客户端频繁触发限流时，立即触发告警，及时排查流量突增的根因，调整配额阈值；
5. **禁止过度限流**：避免将配额阈值设置过低，导致客户端长期处于限流状态，业务延迟大幅升高，甚至出现请求超时；
6. **配合分区隔离使用**：配额管控与 Topic 分区、Broker 节点隔离配合使用，实现核心业务与非核心业务的物理资源隔离，限流作为兜底管控手段。