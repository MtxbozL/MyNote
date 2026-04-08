## 1.4 Kafka 环境搭建

本节覆盖 Kafka 单机环境的两种主流部署方式：二进制包单机部署、Docker/Docker-Compose 快速部署，均基于 3.x 版本主推的 KRaft 架构（去 ZooKeeper 化），同时兼容 2.8 及以下版本的 ZooKeeper 依赖模式。

### 1.4.1 前置依赖说明

- **JDK 依赖**：Kafka 基于 JVM 运行，需提前安装 Java 8 及以上版本，推荐 Java 11（长期支持版本，与 Kafka 3.x 兼容性最优）；
- **操作系统**：Linux 为生产环境首选操作系统，Windows/MacOS 仅用于本地开发测试；
- **硬件要求**：开发测试环境最低配置为 2 核 CPU、4G 内存；生产环境需根据业务吞吐需求完成容量规划。

### 1.4.2 二进制包单机部署（KRaft 模式）

KRaft 模式为 Kafka 3.x 版本官方主推架构，通过内置的 Raft 共识协议实现元数据管理与集群选主，完全去除对 ZooKeeper 的外部依赖，降低了集群运维复杂度。

1. **安装包获取**：从 Apache Kafka 官方镜像站下载最新稳定版二进制安装包，格式为`kafka_2.13-3.7.0.tgz`（2.13 为 Scala 版本号，3.7.0 为 Kafka 版本号，推荐使用最新稳定版）；
2. **解压安装**：执行解压命令`tar -zxvf kafka_2.13-3.7.0.tgz`，将安装包解压至指定目录（如`/opt/kafka`），并进入安装根目录；
3. **集群 ID 生成**：KRaft 模式需生成唯一集群 ID，执行命令`./bin/kafka-storage.sh random-uuid`，获取 UUID 字符串；
4. **存储目录格式化**：使用生成的 UUID 格式化 Kafka 存储目录，执行命令`./bin/kafka-storage.sh format -t <生成的UUID> -c ./config/kraft/server.properties`；
5. **服务启动**：执行命令`./bin/kafka-server-start.sh -daemon ./config/kraft/server.properties`，以守护进程方式启动 Kafka 服务；
6. **服务可用性验证**：执行`jps`命令，若输出中存在 Kafka 进程则服务启动成功；也可通过 1.5 节的 CLI 工具创建 Topic 完成端到端验证。

### 1.4.3 兼容模式：ZooKeeper 依赖单机部署

针对 2.8 及以下版本，Kafka 需依赖 ZooKeeper 实现元数据管理与选主，部署步骤如下：

1. 启动 ZooKeeper 服务：Kafka 安装包内置 ZooKeeper，可执行命令`./bin/zookeeper-server-start.sh -daemon ./config/zookeeper.properties`启动内置 ZK 服务；
2. 修改配置文件：编辑`config/server.properties`，配置`zookeeper.connect=localhost:2181`，指定 ZK 服务地址；
3. 启动 Kafka 服务：执行命令`./bin/kafka-server-start.sh -daemon ./config/server.properties`启动服务；
4. 服务验证：同 KRaft 模式，通过`jps`命令或 CLI 工具验证服务可用性。

### 1.4.4 Docker/Docker-Compose 快速搭建

Docker 部署方式适用于本地开发测试环境，可快速完成环境搭建，无需配置 JDK 等前置依赖，本节基于 Apache 官方镜像提供部署方案。

#### 1.4.4.1 Docker 单节点部署命令

```bash
docker run -d --name kafka-standalone \
  -p 9092:9092 \
  -e KAFKA_NODE_ID=1 \
  -e KAFKA_PROCESS_ROLES=broker,controller \
  -e KAFKA_CONTROLLER_QUORUM_VOTERS=1@localhost:9093 \
  -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093 \
  -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092 \
  -e KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER \
  -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
  -e KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR=1 \
  -e KAFKA_TRANSACTION_STATE_LOG_MIN_ISR=1 \
  apache/kafka:3.7.0
```

#### 1.4.4.2 Docker-Compose 部署

Docker-Compose 可实现服务的配置化管理，便于环境的快速启停与迁移，`docker-compose.yaml`配置文件如下：

```yaml
version: '3.8'
services:
  kafka:
    image: apache/kafka:3.7.0
    container_name: kafka-standalone
    ports:
      - "9092:9092"
    environment:
      # KRaft模式核心配置
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      # 单节点副本配置
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
    volumes:
      - ./kafka-data:/var/lib/kafka/data
    restart: unless-stopped
```

部署命令：

1. 创建`docker-compose.yaml`文件，写入上述配置；
2. 执行`docker-compose up -d`命令启动服务；
3. 执行`docker-compose ps`命令查看服务运行状态，验证启动成功。
