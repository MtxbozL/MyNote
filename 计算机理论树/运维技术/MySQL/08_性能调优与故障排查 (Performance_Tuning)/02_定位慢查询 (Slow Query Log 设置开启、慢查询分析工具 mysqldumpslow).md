
>慢查询是 MySQL 性能问题的核心表象，90% 以上的数据库性能瓶颈都可归因于低效的慢 SQL。慢查询日志（Slow Query Log）是 MySQL 提供的慢 SQL 记录工具，可精准捕获执行时间超过阈值的 SQL 语句，是性能问题定位的核心入口。

### 8.2.1 慢查询日志配置与开启

慢查询日志默认关闭，生产环境推荐永久开启，配置分为**动态临时生效**与**配置文件永久生效**两种方式。

#### 核心配置参数详解

|参数名|取值范围|核心作用|生产环境推荐配置|
|---|---|---|---|
|slow_query_log|ON/OFF|慢查询日志的总开关，控制是否开启慢查询记录|ON，永久开启|
|slow_query_log_file|文件路径|慢查询日志的存储文件路径|MySQL 数据目录下，如`/var/lib/mysql/mysql-slow.log`|
|long_query_time|数值（单位秒）|慢查询阈值，SQL 执行时间超过该值会被记录到慢日志，支持小数精度到毫秒|1 秒，高并发核心业务可设置为 0.5 秒，避免阈值过低导致日志量爆炸|
|log_queries_not_using_indexes|ON/OFF|是否记录未使用索引的 SQL，即使执行时间未超过 long_query_time 阈值|测试环境开启，生产环境谨慎开启；开启后会导致全表扫描的小表查询大量写入日志，占用磁盘空间|
|log_slow_admin_statements|ON/OFF|是否记录 ALTER、CREATE INDEX 等管理语句的慢查询|OFF，生产环境管理语句均在低峰期执行，无需记录|
|log_throttle_queries_not_using_indexes|数值|限制每分钟记录的未使用索引 SQL 的最大数量，避免日志量爆炸|60，开启 log_queries_not_using_indexes 时必须配置|

#### 配置生效方式

1. **动态临时生效**：MySQL 运行期间执行以下 SQL，无需重启服务，重启后配置失效，适用于临时排查场景：
    
    ```sql
    -- 开启慢查询日志
    SET GLOBAL slow_query_log = 'ON';
    -- 设置慢查询阈值为1秒
    SET GLOBAL long_query_time = 1;
    -- 限制未使用索引的SQL记录频率
    SET GLOBAL log_throttle_queries_not_using_indexes = 60;
    ```
    
    > 注意：全局参数修改后，仅对新建立的会话生效，已存在的会话需重新连接才会生效。
    
2. **配置文件永久生效**：修改 MySQL 配置文件`my.cnf`（Linux）或`my.ini`（Windows），在`[mysqld]`节点下添加以下配置，重启 MySQL 服务后永久生效：
    
    ```bash
    [mysqld]
    # 慢查询核心配置
    slow_query_log = ON
    slow_query_log_file = /var/lib/mysql/mysql-slow.log
    long_query_time = 1
    log_throttle_queries_not_using_indexes = 60
    ```

### 8.2.2 慢查询分析工具

慢查询日志生成后，需通过专业工具进行统计分析，从海量日志中定位核心高频慢 SQL，优先解决对业务影响最大的性能瓶颈。

#### 8.2.2.1 mysqldumpslow（官方原生工具）

`mysqldumpslow`是 MySQL 自带的慢查询日志分析工具，无需额外安装，可对慢 SQL 进行归类统计，避免相同 SQL 的不同参数值重复统计，是生产环境最便捷的分析工具。

##### 核心语法与常用参数

```bash
mysqldumpslow [参数] 慢查询日志文件路径
```

|常用参数|核心作用|
|---|---|
|-s|排序规则，指定统计结果的排序维度：<br><br>・t：按总执行时间排序（默认）<br><br>・c：按执行次数排序<br><br>・l：按锁等待时间排序<br><br>・r：按返回行数排序|
|-t N|只显示排名前 N 的结果，避免输出过多内容|
|-g|支持正则表达式过滤，只匹配包含指定关键词的 SQL|
|-a|不将 SQL 中的数值常量、字符串常量替换为 N/S，显示原始 SQL|

##### 典型使用场景

1. 按总执行时间排序，显示 Top 10 慢 SQL：
    
    ```bash
    mysqldumpslow -s t -t 10 /var/lib/mysql/mysql-slow.log
    ```
    
2. 按执行次数排序，显示 Top 20 高频慢 SQL：
    
    ```bash
    mysqldumpslow -s c -t 20 /var/lib/mysql/mysql-slow.log
    ```
    
3. 过滤包含`ORDER BY`的排序类慢 SQL，显示 Top 10：
    
    ```bash
    mysqldumpslow -s t -t 10 -g "ORDER BY" /var/lib/mysql/mysql-slow.log
    ```
##### 输出结果解读

输出结果核心包含以下关键字段，可快速定位 SQL 的性能问题：

- `Count`：该 SQL 的执行总次数；
- `Time`：执行总时间、平均执行时间、最大 / 最小执行时间；
- `Lock`：锁等待总时间、平均锁等待时间；
- `Rows`：返回总行数、平均返回行数；
- 最后为归类后的 SQL 模板，常量被替换为`N`/`S`。

#### 8.2.2.2 进阶分析工具

生产环境海量慢日志场景下，可使用进阶工具实现更精细化的分析：

1. **pt-query-digest**：Percona Toolkit 工具集的核心组件，是业界最常用的慢查询分析工具，可生成详细的 SQL 性能报告，包括执行时间分布、锁等待、IO 开销、优化建议等，支持慢日志、Binlog、TCP 抓包等多数据源分析。
2. **MySQL Workbench**：官方 GUI 工具，内置慢查询日志可视化分析功能，可生成图表化的性能报告，适合新手使用。

### 8.2.3 慢查询分析优先级

面对海量慢 SQL，需遵循以下优先级排序，优先解决对业务影响最大的问题：

1. **高频执行的慢 SQL**：即使单次执行时间仅 1-2 秒，每秒执行数十次，会占用大量数据库 CPU 资源，是核心优化目标；
2. **长执行时间的大事务 SQL**：单次执行时间超过 10 秒的 SQL，会占用连接、持有锁、阻塞其他事务，极易引发业务雪崩；
3. **未使用索引的全表扫描 SQL**：即使当前执行时间短，随着表数据量增长，性能会线性下降，必须提前优化；
4. **锁等待时间过长的 SQL**：锁等待占比超过执行时间 50% 的 SQL，需排查锁冲突与事务设计问题。
