
Rebalance（重平衡）是 Kafka 消费组的核心分布式协调协议，是本章的核心重点。当消费组的成员、订阅关系、分区数量发生变化时，GroupCoordinator 会触发 Rebalance，重新分配订阅 Topic 的全部分区给消费组内的消费者，保证分区分配符合消费组核心铁律，实现负载均衡与故障自动转移。

### 4.5.1 Rebalance 触发条件

Rebalance 的触发条件严格限定为以下三类，无其他触发场景，生产环境的非预期 Rebalance 均由这三类场景的异常情况引发。

#### 1. 消费组成员变更（最常见触发场景）

该场景是生产环境 Rebalance 的主要触发来源，包括正常与异常两类情况：

- **正常变更**：消费者实例扩容（新增消费者）、缩容（下线消费者）、应用滚动发布（消费者实例重启）；
- **异常变更**：消费者实例被 GroupCoordinator 判定为下线，踢出消费组，核心原因包括：
    
    1. 会话超时：消费者在`session.timeout.ms`（默认 10 秒）内未向 Coordinator 发送心跳请求，被判定为宕机；
    2. 消费超时：消费者两次`poll()`调用的间隔超过`max.poll.interval.ms`（默认 5 分钟），被判定为消费能力失效，踢出消费组，这是生产环境非预期 Rebalance 的首要诱因；
    3. 消费者进程异常退出、JVM Full GC 时间过长、网络分区导致心跳与请求无法送达 Coordinator。
    

#### 2. 消费组订阅的 Topic 变更

- 消费者采用正则表达式订阅 Topic，新增了匹配正则规则的 Topic；
- 消费组内的消费者修改了订阅关系，新增或取消了对某个 Topic 的订阅。

#### 3. 订阅 Topic 的分区数量变更

为消费组订阅的 Topic 执行了分区扩容操作，新增的分区需要分配给消费组内的消费者，触发 Rebalance。

### 4.5.2 Coordinator 协调者机制

GroupCoordinator 是运行在每个 Broker 节点上的核心组件，是消费组的 “管理者”，负责消费组的全生命周期管理、Rebalance 的触发与协调、Offset 提交的处理、消费者存活状态的监控。

#### 1. 消费组与 Coordinator 的绑定规则

每个消费组只会绑定一个 GroupCoordinator，绑定规则与 Offset 存储分区强绑定：消费组对应的`__consumer_offsets`分区的 Leader 副本所在的 Broker，即为该消费组的 GroupCoordinator。该绑定关系保证了消费组的所有请求都由同一个 Coordinator 处理，避免了分布式一致性问题。

#### 2. Coordinator 核心职责

1. 消费组成员管理：处理消费者的加入、退出请求，维护消费组的成员列表与存活状态；
2. Rebalance 全流程协调：触发 Rebalance，管理 Rebalance 的状态机，同步分区分配方案；
3. Offset 提交管理：处理消费者的 Offset 提交请求，将 Offset 数据写入`__consumer_offsets`，响应消费者的 Offset 拉取请求；
4. 消费者存活监控：通过心跳机制监控消费者的存活状态，超时未收到心跳则将消费者踢出组，触发 Rebalance；
5. 消费组状态管理：维护消费组的状态机，保证消费组生命周期的一致性。

### 4.5.3 Rebalance 全流程机制

Kafka Rebalance 分为两种模式：**Eager 急切式模式**（对应 Range、RoundRobin、Sticky 分配策略）与**Cooperative 协作式模式**（对应 CooperativeSticky 分配策略），二者的流程与对业务的影响有本质区别。

#### 1. Eager 急切式 Rebalance 全流程

Eager 模式是传统的 Rebalance 模式，核心特征是**Rebalance 触发时，所有消费者必须放弃全部已分配的分区，停止消费，进入全组 STW 状态**，直到 Rebalance 完成。全流程分为 5 个核心阶段：

1. **Join Group 加入组阶段**
    
    - Rebalance 触发后，消费组进入 PreparingRebalance 状态，所有消费者必须停止消费，放弃所有已分配的分区，向 Coordinator 发送 JoinGroup 请求，上报自身的订阅信息、分配策略、客户端 ID；
    - Coordinator 等待所有消费者的 JoinGroup 请求，或等待`max.poll.interval.ms`超时，筛选出消费组内所有消费者支持的统一分配策略，从消费者中选举出**Group Leader**（通常为第一个加入组的消费者）；
    - Coordinator 将所有消费者的订阅信息、成员列表，通过 JoinGroup 响应返回给 Group Leader，其他消费者仅收到基础响应信息。
    
2. **Sync Group 同步分配阶段**
    
    - Group Leader 收到 JoinGroup 响应后，根据配置的分配策略，计算出全部分区的分配方案；
    - Group Leader 向 Coordinator 发送 SyncGroup 请求，将完整的分区分配方案上报给 Coordinator；
    - 其他消费者同时向 Coordinator 发送 SyncGroup 请求，等待分配方案；
    - Coordinator 收到分配方案后，通过 SyncGroup 响应，将每个消费者对应的分区分配结果，返回给对应消费者。
    
3. **分区撤销阶段**
    
    - 消费者收到 SyncGroup 响应后，执行分区撤销逻辑，提交未提交的 Offset，释放所有分区的资源，该阶段全程处于停止消费状态。
    
4. **分区分配阶段**
    
    - 消费者完成分区撤销后，根据分配结果，初始化新分配的分区，拉取对应分区的起始 Offset，注册分区监听器，准备开始消费。
    
5. **Stable 稳定运行阶段**
    
    - 所有消费者完成分区初始化，向 Coordinator 确认就绪，消费组进入 Stable 稳定状态，所有消费者开始正常消费；
    - 消费者定期向 Coordinator 发送心跳请求，上报存活状态，直到下一次 Rebalance 触发。
    

#### 2. Cooperative 协作式 Rebalance 流程优化

Cooperative 模式彻底解决了 Eager 模式的全组 STW 问题，核心优化点包括：

1. 无需全量撤销分区：Rebalance 触发时，消费者无需放弃所有分区，仅需释放需要调整的分区，其余分区全程保持正常消费，不会出现全组消费中断；
2. 分阶段增量调整：采用两轮 Rebalance 协议，第一轮同步成员与订阅信息，计算需要调整的分区；第二轮仅对需要变动的分区进行重新分配，无需全量重新分配；
3. 无 STW 运行：整个 Rebalance 过程中，绝大多数消费者的绝大多数分区都在正常消费，业务完全无感知，仅调整的分区会短暂暂停。

### 4.5.4 Rebalance 的性能影响与规避策略

#### 1. Rebalance 的核心危害

1. **全组消费中断**：Eager 模式下，Rebalance 期间消费组完全停止消费，端到端延迟大幅升高，引发消息积压，流量洪峰场景下极易导致消费雪崩；
2. **消费性能损耗**：分区重新分配后，消费者需要重新拉取分区元数据、重建网络连接、加载消费进度，导致消费延迟升高，吞吐下降；
3. **重复消费风险**：Rebalance 触发时，消费者未提交的 Offset 可能会丢失，重启后会重新消费已处理的消息，引发重复消费；
4. **频繁 Rebalance 会导致消费组长期处于不稳定状态，甚至出现消费停滞，是生产环境消费端的头号风险点**。

#### 2. 非预期 Rebalance 的核心规避策略

生产环境的 Rebalance 优化核心是：**允许正常的扩缩容、发布触发的 Rebalance，彻底杜绝非预期的、频繁的 Rebalance**，从参数调优、代码规范、架构设计三个维度实现。

##### 维度一：核心参数调优

参数调优是规避非预期 Rebalance 的基础，需根据业务场景精准配置，核心参数如下：

|参数名称|默认值|调优建议|核心作用|
|---|---|---|---|
|session.timeout.ms|10000（10 秒）|生产环境建议 6000~30000ms|控制消费者会话超时时间，避免网络抖动、轻微 GC 导致的误判下线；需与心跳间隔配合，建议心跳间隔为该值的 1/3|
|heartbeat.interval.ms|3000（3 秒）|建议为 session.timeout.ms 的 1/3|消费者心跳发送间隔，保证会话超时内至少能发送 3 次心跳，提升容错能力|
|max.poll.interval.ms|300000（5 分钟）|必须设置为业务单批次最大处理时间的 2~3 倍|控制两次 poll () 调用的最大间隔，避免业务处理耗时过长导致消费者被踢出组，是解决非预期 Rebalance 的核心参数|
|max.poll.records|500|根据单条消息处理时间调优，避免单批次处理时间过长|单次 poll () 拉取的最大消息数，控制单批次处理的消息量，保证处理时间不超过 max.poll.interval.ms|
|partition.assignment.strategy|3.0 + 默认 CooperativeStickyAssignor|生产环境强制使用 CooperativeStickyAssignor|彻底消除 Rebalance 的全组 STW 问题，降低 Rebalance 对业务的影响|

##### 维度二：代码规范与最佳实践

1. **消费者优雅关闭**：注册 JVM 关闭钩子，在应用关闭时主动调用`consumer.wakeup()`和`consumer.close()`方法，主动通知 Coordinator 消费者下线，立即触发 Rebalance，无需等待会话超时，减少停服时间与重复消费风险；
2. **禁止在 poll () 线程中执行耗时操作**：业务处理逻辑、远程调用、数据库操作等耗时逻辑，必须异步化处理，避免阻塞 poll () 线程，导致两次 poll () 间隔超时；
3. **完善异常处理机制**：业务处理异常必须捕获，不能因为单条消息处理异常导致 poll () 循环终止，消费者停止拉取被踢出组；
4. **避免频繁的消费者启停**：应用滚动发布时采用分批发布，减少消费组成员变更的频率，降低 Rebalance 触发次数。

##### 维度三：架构设计优化

1. **消费者数量控制**：消费组内的消费者数量，不得超过订阅 Topic 的总分区数，多余的消费者会处于闲置状态，同时增加 Rebalance 的复杂度，提升非预期 Rebalance 的概率；
2. **固定 Topic 订阅**：尽量避免使用正则表达式订阅 Topic，防止新增匹配 Topic 触发非预期 Rebalance，采用固定 Topic 名称订阅；
3. **减少分区扩容操作**：提前规划 Topic 的分区数量，尽量避免生产环境的分区扩容，减少 Rebalance 触发；
4. **核心业务隔离**：核心业务消费组采用独立的 Topic，避免与非核心业务共用 Topic，减少无关变更对核心消费组的影响。