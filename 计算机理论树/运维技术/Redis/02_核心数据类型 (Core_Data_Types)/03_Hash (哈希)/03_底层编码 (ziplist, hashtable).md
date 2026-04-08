Hash 的底层编码分为 **ziplist（压缩列表）**和**hashtable（哈希表 / 字典）** 两种，Redis 会根据 Hash 的 field 数量与 field/value 长度自动选择编码，满足阈值时自动转换，转换过程对用户无感知。

#### 一、ziplist 压缩列表编码

- **适用场景**：当 Hash 同时满足以下两个条件时，使用 ziplist 编码：
    
    1. Hash 中的 field-value 对总数量不超过`hash-max-ziplist-entries`（redis.conf 配置，默认 512）；
    2. Hash 中每个 field 和每个 value 的字节长度均不超过`hash-max-ziplist-value`（默认 64 字节）。
    
- **底层存储规则**：ziplist 中，每两个连续的 entry 成对存储 field 和 value，即 field-entry 紧跟 value-entry，所有 field-value 对紧凑存储在同一段连续内存中。
- **核心优势**：
    
    1. 内存占用极低，无指针冗余，内存利用率远高于 hashtable；
    2. 连续内存空间 CPU 缓存命中率高，小 field 数量场景下读写性能优异；
    3. 无哈希冲突问题，存储结构简单。
    
- **核心缺陷**：
    
    1. 读写操作需遍历 ziplist 查找对应 field，时间复杂度 O (N)，field 数量增多时性能急剧下降；
    2. 插入 / 删除操作会触发内存重分配，存在连锁更新问题，大 field 场景性能极差；
    3. 不支持高效随机读写，仅适合小 field 数量、小元素长度的场景。
    

#### 二、hashtable 哈希表（字典）编码

- **适用场景**：当 Hash 不满足 ziplist 的任一条件时，自动从 ziplist 转换为 hashtable 编码，**转换后不可逆**，即使后续 field 数量减少、元素长度缩短，也不会转回 ziplist。
- **底层结构**：与 Redis 全局 Key 空间使用的哈希表完全一致，基于字典（dict）结构实现，核心组成如下：
    
    1. 字典管理结构`dict`：包含两个哈希表`ht[0]`和`ht[1]`，`ht[0]`为主哈希表，`ht[1]`仅用于 rehash；`rehashidx`记录 rehash 进度，-1 表示未进行 rehash；
    2. 哈希表结构`dictht`：包含哈希桶数组`table`、哈希表大小`size`（2 的整数次幂）、大小掩码`sizemask`（size-1，用于计算哈希索引）、已使用元素数量`used`；
    3. 哈希桶节点`dictEntry`：包含 field 键指针、value 值指针、next 指针（指向同一哈希桶的下一个节点，采用链地址法解决哈希冲突）。
    
- **核心原理**：
    
    1. 哈希计算：对 field 计算哈希值，通过`sizemask`计算哈希桶索引，定位到对应哈希桶；
    2. 哈希冲突解决：采用链地址法，同一哈希桶的多个节点通过单向链表串联；
    3. 渐进式 rehash：当哈希表负载因子（used/size）超过阈值（默认 1）时触发扩容，扩容后大小为当前 used 的 2 倍（2 的整数次幂）；为避免一次性 rehash 阻塞主线程，将 rehash 操作分散到每次哈希表的增删改查操作中，同时后台定时任务辅助执行，全程无主线程阻塞。
    
- **核心优势**：
    
    1. 读写操作平均时间复杂度 O (1)，性能稳定，不受 field 数量影响，适合大 field 数量、大元素长度场景；
    2. 支持高效随机读写，无遍历开销；
    3. 渐进式 rehash 机制，避免扩容阻塞主线程；
    4. 无连锁更新问题，插入删除性能稳定。
    
- **核心缺陷**：
    
    1. 内存开销大，每个 dictEntry 节点存在 3 个指针冗余，小 field 场景内存利用率远低于 ziplist；
    2. 哈希冲突会导致链表过长，查询性能下降，Redis 通过 rehash 自动调整哈希表大小，降低冲突概率。
    

#### 生产环境编码优化规范

1. 合理设置`hash-max-ziplist-entries`与`hash-max-ziplist-value`，默认 512 与 64 字节为性能与内存的最优平衡，小元素场景可适当调高，大元素场景可适当调低，避免频繁编码转换；
2. 单个 Hash 的 field 数量建议控制在 1000 以内，尽量使用 ziplist 编码，提升内存利用率；
3. 禁止在 Hash 的 field/value 中存储超过 64 字节的超大内容，会触发编码转换为 hashtable，内存占用大幅上升；
4. 大哈希必须使用 HSCAN 增量遍历，绝对禁止使用 HGETALL、HKEYS、HVALS，避免阻塞主线程；
5. 结构化对象存储优先使用 Hash，相比 String 序列化存储，按需读写的网络开销更低，原子性更新更安全。