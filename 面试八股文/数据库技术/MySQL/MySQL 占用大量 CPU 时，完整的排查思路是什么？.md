#### 核心标准答案

MySQL CPU 占用过高，90% 以上的场景都是 SQL 执行问题导致的，我会遵循「**先定位 SQL，再分析根因，最后优化解决**」的思路，完整排查流程如下：

1. **第一步：确认整体状态，锁定 MySQL 进程**
    
    - 执行`top`命令，确认 mysqld 进程的 CPU 使用率，区分是用户态 CPU（% us）高，还是内核态 CPU（% sy）高；用户态 CPU 高，大概率是 SQL 执行、逻辑运算导致；内核态 CPU 高，大概率是磁盘 IO、网络 IO、系统调用导致；
    - 执行`pidstat -u -p mysqld_pid 1`，持续监控 MySQL 进程的 CPU 占用变化，确认是突发升高，还是持续升高。
    
2. **第二步：定位导致 CPU 高的慢 SQL（核心步骤）**
    
    这是排查的核心，90% 的 MySQL CPU 高都是慢 SQL 导致的，核心定位方式：
    
    - **实时定位正在执行的 SQL**：
        
        登录 MySQL，执行`show processlist;`，或查询`information_schema.processlist`表，查看当前正在执行的 SQL，重点关注`Time`（执行时间）长、`State`为`Sending data`、`Copying to tmp table`、`Sorting result`的 SQL，这些就是导致 CPU 高的元凶。
    - **通过慢查询日志定位**：
        
        1. 确认慢查询日志已开启：执行`show variables like 'slow_query_log';`，如果未开启，临时开启`set global slow_query_log = 'ON';`，设置阈值`set global long_query_time = 1;`（超过 1 秒的 SQL 记录，可根据场景调小到 0.5 秒）；
        2. 通过`pt-query-digest`/`mysqldumpslow`工具分析慢查询日志，按执行次数、总执行时间排序，找到 Top 慢 SQL。
        
    
3. **第三步：分析慢 SQL 的执行计划，定位根因**
    
    拿到慢 SQL 后，执行`explain 慢SQL语句`，查看执行计划，定位 SQL 的性能问题，核心关注字段：
    
    - **type 字段**：访问类型，如果结果是`ALL`，代表全表扫描，没有命中索引，是 CPU 高的最常见原因；理想状态至少是`range`级别，最优是`ref`、`eq_ref`、`const`；
    - **key 字段**：实际命中的索引，如果为 NULL，代表没有使用索引，需要优化索引；
    - **rows 字段**：MySQL 为了找到结果需要扫描的行数，扫描行数远大于返回行数，代表索引效率极低，需要优化；
    - **Extra 字段**：
        
        - 出现`Using filesort`：代表 SQL 需要额外的排序操作，无法通过索引完成排序，会大量消耗 CPU；
        - 出现`Using temporary`：代表 SQL 使用了临时表，通常是 GROUP BY、UNION 导致，会消耗大量 CPU 和内存；
        - 出现`Using where`：代表使用了 WHERE 过滤，但没有走索引，需要优化。
            
            常见的 SQL 根因：
        
    - 索引失效，导致全表扫描、全索引扫描；
    - SQL 中包含大量的计算、函数、JOIN 操作，导致 CPU 运算量过大；
    - 排序、分组操作无法通过索引完成，出现文件排序、临时表；
    - 高频执行的简单 SQL，没有命中索引，即使单次执行耗时短，高频执行也会导致 CPU 占用飙升。
    
4. **第四步：排查其他非 SQL 的根因**
    
    如果 SQL 没有问题，排查其他场景：
    
    - **事务与锁问题**：查看`show engine innodb status;`，是否有大量的锁等待、死锁，导致事务频繁重试、CPU 占用升高；
    - **MySQL 参数配置不合理**：比如`innodb_buffer_pool_size`设置过小，导致频繁的磁盘 IO；`thread_cache_size`设置过小，导致频繁创建销毁线程；`join_buffer_size`、`sort_buffer_size`设置不合理，导致大量的内存分配和释放；
    - **连接数过多**：大量的数据库连接，导致 MySQL 的线程调度开销过大，CPU 占用升高；
    - **硬件瓶颈**：磁盘 IO 性能不足，导致 SQL 等待 IO，线程阻塞，CPU 占用升高；内存不足，导致频繁的 swap 交换，CPU 占用升高；
    - **MySQL bug**：比如特定版本的 MySQL 优化器 bug，导致 SQL 执行计划错误，出现性能问题。
    
5. **第五步：针对性优化，验证效果**
    
    根据根因针对性优化：
    
    - 索引优化：为慢 SQL 创建合适的索引，优化联合索引，修复索引失效的场景；
    - SQL 优化：改写 SQL，避免在索引字段上使用函数、避免大表 JOIN、优化排序分组操作、拆分复杂 SQL，避免文件排序和临时表；
    - 参数优化：调整 MySQL 的内存参数、线程参数、缓冲池参数，适配服务器硬件；
    - 架构优化：读写分离、分库分表、增加缓存层，减少 MySQL 的查询压力。
        
        优化完成后，再次查看 MySQL 的 CPU 使用率、SQL 执行时间，确认优化生效，CPU 占用恢复正常。
    

#### 专家级拓展

- 生产环境高频根因 Top3：
    
    1. 索引失效，全表扫描，占比超过 70%；
    2. SQL 包含`Using filesort`、`Using temporary`，排序和临时表导致 CPU 飙升；
    3. 高频小 SQL，没有索引，单次执行快，但每秒执行上万次，累计 CPU 占用极高。
    
- 进阶排查工具：
    
    1. `performance_schema`：MySQL 内置的性能监控库，可查看 SQL 的执行耗时、扫描行数、CPU 占用，精准定位高频慢 SQL；
    2. `show profile`：查看 SQL 执行的每个阶段的 CPU、IO 耗时，定位 SQL 的性能瓶颈在哪个阶段；
    3. Prometheus+Grafana：监控 MySQL 的 CPU 使用率、连接数、QPS、慢查询数量、锁等待时间，提前发现异常。
    
- 生产环境最佳实践：
    
    1. 上线前必须用 explain 分析 SQL 的执行计划，禁止无索引的 SQL 上线；
    2. 开启慢查询日志，定期分析慢 SQL，提前优化，避免 CPU 飙升；
    3. 合理设置`long_query_time`，生产环境建议设置为 0.5 秒，甚至 0.2 秒，提前发现潜在的慢 SQL；
    4. 配置 MySQL 的 CPU 使用率监控告警，提前发现异常。
    

#### 面试避坑指南

- 严禁只说「优化 SQL、加索引」，不说完整的排查流程，会被面试官判定为无体系化排查能力；
- 避免一上来就调整 MySQL 参数、升级服务器配置，90% 的场景都是 SQL 和索引问题，优先定位 SQL 根因；
- 不要只看 SQL 的执行时间，忽略高频执行的小 SQL，很多时候 CPU 高不是单条慢 SQL，而是每秒执行上万次的无索引小 SQL。