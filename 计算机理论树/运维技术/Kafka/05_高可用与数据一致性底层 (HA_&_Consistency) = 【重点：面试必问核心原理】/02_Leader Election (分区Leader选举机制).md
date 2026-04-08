
分区 Leader 选举是 Controller 的核心职责，是 Kafka 实现分区级高可用的核心机制。每个分区同一时间只能有一个 Leader 副本，负责处理该分区的所有读写请求；Follower 副本仅负责同步 Leader 数据，不对外提供任何读写服务，该设计简化了分布式一致性模型，避免了主从副本的数据不一致问题。

### 5.2.1 选举核心前置铁律

Kafka 分区 Leader 选举的核心前提，是保障数据不丢失，因此制定了不可突破的铁律：**只有处于 ISR（同步副本集合）中的副本，才有资格被选举为新的 Leader**，除非显式开启 Unclean Leader Election（脏选举）。

### 5.2.2 选举触发场景

所有分区 Leader 选举动作必须由 Active Controller 统一触发，禁止 Broker 自主发起，保证集群选主规则的全局一致性，核心触发场景包括：

1. Broker 宕机 / 下线：该 Broker 上承载 Leader 副本的分区，需要重新选举 Leader；
2. Broker 上线：副本重新加入 ISR 集合，Controller 触发优先副本选举，恢复分区的默认 Leader 分布，实现集群负载均衡；
3. Topic 创建 / 分区扩容：为新创建的分区选举初始 Leader，完成分区初始化；
4. 副本重分配：执行分区副本迁移后，为分区选举新的 Leader，完成副本切换；
5. ISR 集合变更：原 Leader 副本被踢出 ISR 集合，需要选举新的 Leader 保障服务可用；
6. 手动触发：用户通过 CLI 工具手动触发分区 Leader 重选举，调整集群负载分布。

### 5.2.3 标准选举流程与算法

Kafka 分区 Leader 选举采用**优先副本 + ISR 有序列表**的确定性算法，保证选举结果的可预期性与负载均衡能力，完整流程如下：

1. Controller 获取该分区的 AR（Assigned Replicas，分配的副本列表），AR 列表的顺序是 Topic 创建时指定的优先副本顺序，第一个副本为**Preferred Replica（优先副本）**，是分区的默认 Leader；
2. Controller 获取该分区当前的 ISR 集合，过滤出其中处于存活状态的副本；
3. 按照 AR 列表的原始顺序，从存活的 ISR 副本中，选出第一个副本作为新的 Leader；
4. 剩余的存活 ISR 副本作为 Follower，Controller 更新分区的 Leader&ISR 信息，同步给集群所有 Broker 节点；
5. 同时更新分区的 Leader Epoch，将任期号 + 1，标识新一轮 Leader 任期的开始，用于后续的数据一致性校验。

### 5.2.4 特殊场景的选举机制

1. **优先副本选举（Preferred Replica Election）**
    
    生产环境中，Broker 宕机、重启会导致分区 Leader 分布不均，少数 Broker 承载大量 Leader 副本，出现负载过载。优先副本选举的核心目标，是将分区的 Leader 恢复为 AR 列表中的第一个优先副本，实现集群读写负载的均匀分布。
    
    触发方式分为两种：自动平衡（`auto.leader.rebalance.enable=true`，默认开启，Controller 定期检测并执行）与手动触发（通过 CLI 工具批量执行）。
    
2. **离线分区的选举**
    
    当分区的 ISR 集合中所有副本都处于离线状态时，分为两种处理逻辑：
    
    - 关闭 Unclean Leader Election（默认）：无法选举新的 Leader，该分区处于离线状态，读写服务不可用，优先保障数据一致性；
    - 开启 Unclean Leader Election：从 OSR（不同步副本集合）中选举存活的副本作为新 Leader，牺牲数据一致性换取服务可用性，详细机制见 5.5 节。