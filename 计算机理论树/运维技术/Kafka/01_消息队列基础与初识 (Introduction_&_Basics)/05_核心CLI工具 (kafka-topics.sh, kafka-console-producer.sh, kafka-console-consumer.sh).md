## 1.5 Kafka 核心 CLI 工具

Kafka 的核心 CLI 工具均位于安装根目录的`bin/`路径下（Windows 系统位于`bin/windows/`路径下），是 Kafka 集群管理、生产消费验证、问题排查的核心工具。3.x 版本 CLI 工具统一通过`--bootstrap-server`参数指定 Broker 地址，替代旧版本的`--zookeeper`参数，兼容 KRaft 与 ZK 模式。

### 1.5.1 kafka-topics.sh

该工具是 Kafka Topic 全生命周期管理的核心工具，支持 Topic 的创建、查看、修改、删除、元数据描述等全量操作。

#### 核心常用命令与参数说明

1. **创建 Topic**

```bash
./bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --topic test_topic --partitions 3 --replication-factor 1
```

关键参数：

- `--create`：指定操作为 Topic 创建；
- `--bootstrap-server`：指定 Kafka Broker 的访问地址；
- `--topic`：指定待创建的 Topic 名称，集群内全局唯一；
- `--partitions`：指定 Topic 的分区数量，决定 Topic 的并行处理能力，创建后仅支持扩容不支持缩容；
- `--replication-factor`：指定每个分区的副本数量，单节点集群不可超过 1，生产环境建议≥2，最大值不可超过集群 Broker 节点数。

2. **查看集群 Topic 列表**

```bash
./bin/kafka-topics.sh --list --bootstrap-server localhost:9092
```

参数`--list`：输出集群中所有可用的 Topic 名称。

3. **查看 Topic 详细元数据**

```bash
./bin/kafka-topics.sh --describe --bootstrap-server localhost:9092 --topic test_topic
```

参数`--describe`：输出 Topic 的分区数、副本因子、每个分区的 Leader 节点、副本分布、ISR 集合等核心元数据；不指定`--topic`则输出集群所有 Topic 的详情。

4. **Topic 分区扩容**

```bash
./bin/kafka-topics.sh --alter --bootstrap-server localhost:9092 --topic test_topic --partitions 6
```

参数`--alter`：指定修改操作；注意：Kafka 仅支持分区数扩容，不支持缩容，缩容会导致分区数据丢失与消费异常。

5. **删除 Topic**

```bash
./bin/kafka-topics.sh --delete --bootstrap-server localhost:9092 --topic test_topic
```

参数`--delete`：删除指定 Topic，需 Broker 配置`delete.topic.enable=true`（默认开启）方可生效。

### 1.5.2 kafka-console-producer.sh

该工具是控制台生产者工具，用于向指定 Topic 发送消息，快速验证生产链路可用性、Topic 配置正确性，主要用于开发测试与问题排查。

#### 核心常用命令与参数说明

1. **基础交互式消息生产**

```bash
./bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic test_topic
```

执行命令后进入交互式控制台，每行输入一条消息，按回车键即可将消息发送至指定 Topic。

2. **带 Key 的消息生产**

```bash
./bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic test_topic --property parse.key=true --property key.separator=,
```

关键参数：

- `--property parse.key=true`：开启消息 Key 解析；
- `--property key.separator=,`：指定 Key 与 Value 的分隔符为英文逗号，输入格式为`<key>,<value>`，例如`user_001,hello kafka`。

3. **核心可选参数**

- `--compression-type`：指定消息压缩类型，可选值`none`、`gzip`、`snappy`、`lz4`、`zstd`，用于验证生产者压缩配置；
- `--acks`：指定生产者 acks 级别，可选值`0`、`1`、`all`，验证消息可靠性配置；
- `--batch-size`：指定生产者批量发送的消息字节数，验证批量发送机制。

### 1.5.3 kafka-console-consumer.sh

该工具是控制台消费者工具，用于从指定 Topic 消费消息，验证消费链路可用性、消息内容正确性，查看历史消息，主要用于开发测试与问题排查。

#### 核心常用命令与参数说明

1. **实时消费新消息**

```bash
./bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test_topic
```

执行命令后进入监听模式，实时打印生产者新发送的消息，默认从 Topic 最新的 Offset 开始消费。

2. **从头消费全量历史消息**

```bash
./bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test_topic --from-beginning
```

参数`--from-beginning`：指定从 Topic 的最早可用 Offset 开始消费，而非最新 Offset，用于查看 Topic 内所有历史消息。

3. **消费消息并打印元数据**

```bash
./bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test_topic --property print.key=true --property print.timestamp=true --property print.offset=true --property key.separator=,
```

关键参数：

- `--property print.key=true`：打印消息 Key；
- `--property print.timestamp=true`：打印消息生产时间戳；
- `--property print.offset=true`：打印消息对应的分区 Offset；
- `--property key.separator=,`：指定 Key 与 Value 的输出分隔符。

4. **指定消费组消费**

```bash
./bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test_topic --group test_consumer_group
```

参数`--group`：指定消费者所属的消费组，同一消费组内的消费者遵循单播消费规则，不同消费组之间为多播消费；若不指定该参数，工具会自动生成随机消费组。

5. **核心可选参数**

- `--partition`：指定消费的分区编号，仅消费该分区的消息；
- `--offset`：指定消费的起始 Offset，可选值`earliest`、`latest`、具体数值，精准控制消费起始位置；
- `--max-messages`：指定最大消费消息条数，消费完成后自动退出，无需手动终止。

---

## 本章小结

本章系统阐述了消息队列的核心价值与基础理论，完成了四款主流消息中间件的技术特性对比与选型分析，明确了 Kafka 的核心定位、设计目标与应用场景，落地了 Kafka 单机环境的两种主流部署方式，并掌握了 Kafka Topic 管理、生产、消费三大核心 CLI 工具的使用方法。本章内容是 Kafka 核心架构、生产消费机制、底层存储原理学习的前置基础，下一章将深入讲解 Kafka 的核心架构与基本组件，构建 Kafka 的全局概念体系。