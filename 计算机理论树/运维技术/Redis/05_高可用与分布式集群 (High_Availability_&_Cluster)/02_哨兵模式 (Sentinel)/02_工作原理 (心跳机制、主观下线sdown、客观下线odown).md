#### 一、心跳检测机制

Sentinel 通过两种核心心跳机制，实现对集群节点的持续监控，是故障检测的基础：

1. **PING 心跳检测**
    
    每个 Sentinel 节点，每隔 1 秒会向所有主节点、从节点、其他 Sentinel 节点发送一次`PING`命令，根据节点的响应判断存活状态：
    
    - 节点在`down-after-milliseconds`配置的时间内，返回有效响应，判定为存活；
    - 节点在`down-after-milliseconds`时间内，未返回有效响应，判定为主观下线。
    
2. **INFO 命令信息同步**
    
    每个 Sentinel 节点，每隔 10 秒会向所有主节点、从节点发送`INFO`命令，获取主从拓扑信息、节点 runid、复制偏移量等核心信息，用于更新集群视图，同时发现新加入的从节点，自动纳入监控范围。
    
3. **PUB/SUB 消息同步**
    
    所有 Sentinel 节点会订阅主节点的`__sentinel__:hello`频道，每个 Sentinel 每隔 2 秒会向该频道发布自身的信息、主节点的配置信息，其他 Sentinel 节点通过订阅该频道，发现新的 Sentinel 节点，同时同步主节点的配置信息，实现 Sentinel 集群的状态共识。
    

#### 二、主观下线（SDOWN, Subjectively Down）

主观下线是**单个 Sentinel 节点**对节点故障的单方面判定，核心规则：

- 当 Sentinel 节点向目标节点发送`PING`命令后，目标节点在`down-after-milliseconds`配置的超时时间内，始终未返回有效响应，该 Sentinel 会将目标节点标记为**主观下线（SDOWN）**；
- 主观下线仅代表单个 Sentinel 的判定，不代表节点真的故障，可能是网络分区、单 Sentinel 节点网络波动导致的误判；
- 从节点、Sentinel 节点的主观下线，不会触发后续的故障转移流程，仅主节点的主观下线，会进入客观下线的判定流程。

#### 三、客观下线（ODOWN, Objectively Down）

客观下线是**Sentinel 集群对主节点故障的集体共识判定**，是触发故障转移的唯一前提，核心规则：

1. 当某个 Sentinel 将主节点标记为 SDOWN 后，会向其他所有 Sentinel 节点发送`SENTINEL is-master-down-by-addr`命令，询问其他 Sentinel 是否也认为该主节点已下线；
2. 其他 Sentinel 节点收到命令后，会根据自身的监控结果，返回 YES/NO 的判定结果；
3. 发起询问的 Sentinel，收到的**YES 票数达到配置的 quorum（法定票数）** 时，会将该主节点标记为**客观下线（ODOWN）**，正式触发故障转移流程；
4. 若未达到法定票数，会取消主节点的 SDOWN 标记，判定为误判。

**核心生产规范**：quorum 法定票数必须设置为**Sentinel 节点数量的半数 + 1**，例如 3 个 Sentinel 节点，quorum 设为 2；5 个 Sentinel 节点，quorum 设为 3，保证只有多数 Sentinel 同意，才能判定主节点客观下线，避免网络分区导致的脑裂问题。
