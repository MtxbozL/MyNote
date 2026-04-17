
>GTID（Global Transaction Identifier，全局事务 ID）是 MySQL 5.6 版本引入的复制增强特性，8.0 版本已完全成熟，彻底替代了传统的基于 Binlog 文件名 + Position 的复制模式，极大简化了主从复制的运维、故障切换与状态维护，是现代 MySQL 主从架构的标准配置。

### 9.4.1 GTID 核心定义与组成

GTID 是一个全局唯一的事务标识，每个事务在主库提交时都会生成一个唯一的 GTID，与事务一一对应，GTID 在整个主从集群中全局唯一，不会重复。

- **GTID 格式**：`GTID = source_id:transaction_id`
    
    1. `source_id`：事务生成节点的`server_uuid`，是 MySQL 实例启动时自动生成的 128 位全局唯一 ID，存储在数据目录的`auto.cnf`文件中，唯一标识一个 MySQL 实例；
    2. `transaction_id`：事务在该实例上的提交序号，是一个严格递增的整数，每个事务对应一个唯一序号。
    
- **GTID 集合**：多个 GTID 组成的集合，用`GTID_SET`表示，用于标识一个实例已执行完成的所有事务，典型格式为：`3E11FA47-71CA-11E1-9E33-C80AA9429562:1-1000`，表示该实例已执行了该 source_id 下 1 到 1000 的所有事务。

### 9.4.2 GTID 复制的核心优势

对比传统的 Binlog 文件名 + Position 复制模式，GTID 复制有不可替代的核心优势：

1. **主从故障切换极度简化**：传统模式下，主库宕机后，从库提升为主库时，其他从库需要重新计算新主库的 Binlog 位置点，极易出错；GTID 模式下，从库只需向新主库发送自身的 GTID 集合，新主库自动识别缺失的事务并补全，无需手动计算位置点，实现自动化主从切换；
2. **数据一致性强保障**：GTID 保证一个事务在一个集群中仅会被执行一次，已执行过的事务会自动跳过，避免重复执行事务导致的主从数据不一致；
3. **复制状态易维护**：通过 GTID 集合可快速判断主从数据是否一致，无需对比 Binlog 文件名与位置点，运维难度大幅降低；
4. **完美适配高可用架构**：MHA、InnoDB Cluster 等高可用方案均优先适配 GTID 复制，故障切换的成功率与准确性远高于传统模式；
5. **支持无主键表的并行复制**：MySQL 8.0 基于 GTID 优化了并行复制逻辑，进一步提升了回放性能。

### 9.4.3 GTID 的生命周期

1. **事务生成与 GTID 分配**：主库执行事务提交时，先为该事务分配一个唯一的 GTID，写入 Binlog 文件的开头，再记录事务的具体内容；
2. **Binlog 传输与写入**：主库将包含 GTID 的 Binlog 发送给从库，从库 IO 线程接收后写入 Relay Log；
3. **GTID 预检查**：从库 SQL 线程读取到 GTID 后，先检查该 GTID 是否已在本地执行过，若已执行，自动跳过该事务；若未执行，继续执行后续的事务内容；
4. **事务执行与 GTID 持久化**：从库执行该事务，将 GTID 与事务内容一起写入本地的 Binlog，同时更新本地的 GTID 集合；
5. **循环同步**：主库持续生成新的 GTID 事务，从库持续同步执行，保证主从 GTID 集合最终一致。

### 9.4.4 生产环境核心配置

开启 GTID 复制需要在主从节点配置统一的核心参数，MySQL 8.0 默认开启 GTID，5.7 版本需手动配置：

```bash
# 主从节点统一配置
gtid_mode = ON # 开启GTID模式
enforce_gtid_consistency = ON # 强制GTID一致性，禁止执行不支持GTID的语句
binlog_format = ROW # 强制使用ROW格式Binlog
log_bin = ON # 开启Binlog
log_slave_updates = ON # 从库将回放的事务写入本地Binlog，级联复制必备，GTID模式强制开启
server_id = 1-2^32-1 # 主从节点必须配置唯一的server_id，严禁重复
```

### 9.4.5 GTID 复制的核心限制与避坑

开启 GTID 后，MySQL 会禁止执行不支持 GTID 一致性的语句，避免数据不一致，核心限制如下：

1. 禁止使用`CREATE TABLE ... SELECT`语句，该语句会拆分为两个事务，无法保证 GTID 的原子性；
2. 禁止在事务中使用`CREATE TEMPORARY TABLE`、`DROP TEMPORARY TABLE`创建临时表；
3. 不支持非事务型存储引擎（如 MyISAM），无法保证事务的原子性与 GTID 一致性；
4. 不支持`SQL_SLAVE_SKIP_COUNTER`跳过事务，需通过注入空事务的方式跳过异常事务。
