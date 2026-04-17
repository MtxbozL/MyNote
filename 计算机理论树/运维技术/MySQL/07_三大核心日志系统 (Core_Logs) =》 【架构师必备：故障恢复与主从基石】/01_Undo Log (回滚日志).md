
>MySQL 的三大核心日志是事务 ACID 特性的底层物理支撑，也是高可用架构、数据备份恢复、主从复制的核心载体。三大日志按所属层级分为两类：**InnoDB 存储引擎层日志（Undo Log、Redo Log）** 与**MySQL Server 层日志（Binlog）**。其中 Undo Log 保障事务原子性、支撑 MVCC 多版本控制；Redo Log 保障事务持久性、实现崩溃安全；Binlog 实现数据归档备份与主从数据同步。三者通过两阶段提交机制协同工作，共同保证 MySQL 的数据一致性与高可用能力。

## 7.1 Undo Log（回滚日志）

### 7.1.1 核心定位与本质特性

- **所属层级**：InnoDB 存储引擎层，仅 InnoDB 支持
- **日志类型**：逻辑日志，记录数据修改的反向操作逻辑，而非物理页的字节级修改
- **两大核心作用**：
    
    1. 保障事务的**原子性**：事务执行失败或执行`ROLLBACK`时，通过 Undo Log 将数据恢复到事务开始前的状态，实现事务回滚；
    2. 支撑**MVCC 多版本并发控制**：作为数据历史版本链的物理载体，为一致性快照读提供可见的历史版本数据，实现无锁并发读。
    
- **逻辑日志本质**：Undo Log 不记录数据页的物理变化，而是记录与修改操作相反的 SQL 逻辑。例如`INSERT`操作对应生成`DELETE`反向逻辑，`UPDATE`操作对应生成还原旧值的`UPDATE`反向逻辑，`DELETE`操作对应生成`INSERT`反向逻辑，回滚时直接执行反向逻辑即可恢复数据。

### 7.1.2 Undo Log 分类与存储结构

#### 7.1.2.1 日志分类

根据操作类型，Undo Log 分为两大类，二者的生命周期与清理规则存在本质差异：

1. **Insert Undo Log**：事务执行`INSERT`操作时生成，仅用于事务回滚。由于`INSERT`插入的新数据仅对当前事务可见，不会被其他事务的快照读引用，因此事务提交后，该 Undo Log 可被直接删除或重用，无需保留历史版本。
2. **Update Undo Log**：事务执行`UPDATE`或`DELETE`操作时生成，既用于事务回滚，也用于支撑 MVCC 的一致性读。事务提交后，该日志不会立即删除，会被保留到 Purge 线程确认没有任何活跃事务需要引用该历史版本后，才会被清理回收。

#### 7.1.2.2 存储结构

InnoDB 通过**回滚段（Rollback Segment）** 管理 Undo Log，核心存储结构如下：

1. 回滚段内包含 1024 个 Undo Slot（Undo 槽），每个 Undo Slot 对应一个独立的 Undo Log 链表，每个事务仅占用一个 Undo Slot；
2. MySQL 5.7 + 版本支持独立 Undo 表空间，默认创建`undo_001`、`undo_002`两个独立表空间文件，摆脱了早期版本 Undo Log 存储在共享表空间`ibdata1`中导致的空间膨胀无法回收的问题；
3. 支持 Undo 表空间的自动截断（Truncate），当 Undo 空间使用率超过阈值且无事务引用历史版本时，自动收缩空间，避免磁盘空间持续占用。

### 7.1.3 Undo Log 生命周期与清理机制

#### 完整生命周期

1. 事务启动，执行 DML 操作时，先生成对应的 Undo Log 记录，再修改 Buffer Pool 中的数据页；
2. Undo Log 会同步生成对应的 Redo Log，保障 Undo Log 本身的崩溃安全；
3. 事务提交时，将该事务的 Undo Log 标记为待清理状态，不会立即删除；
4. InnoDB 后台 Purge 线程定期扫描，判断 Undo Log 是否被任何活跃事务的 Read View 引用；
5. 无任何事务引用的 Undo Log，会被 Purge 线程清理或放入空闲链表重用；仍被引用的 Undo Log 会继续保留，直到所有依赖的事务结束。

#### 核心配置参数

|参数名|核心作用|推荐配置|
|---|---|---|
|innodb_undo_tablespaces|独立 Undo 表空间的数量|2-4 个，避免单文件过大|
|innodb_undo_log_truncate|是否开启 Undo 表空间自动截断|ON，生产环境开启|
|innodb_max_undo_log_size|Undo 表空间自动截断的阈值|1GB，避免单文件膨胀|
|innodb_purge_threads|Purge 线程的数量|4-8 个，高并发写场景调大|

### 7.1.4 常见误区澄清

1. 误区：Undo Log 是 Redo Log 的备份。
    
    纠正：二者作用完全独立，Undo Log 负责回滚与 MVCC，Redo Log 负责崩溃恢复，不存在备份关系。
2. 误区：Undo Log 仅存储回滚所需的数据。
    
    纠正：Undo Log 是 MVCC 版本链的物理载体，是一致性快照读的核心依赖，其生命周期不仅受事务提交影响，还受活跃事务的快照读引用影响。
