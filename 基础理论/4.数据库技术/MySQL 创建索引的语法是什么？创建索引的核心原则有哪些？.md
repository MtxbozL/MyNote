#### 核心标准答案

##### 一、索引创建的核心语法

MySQL 支持多种类型的索引，创建语法分为「建表时创建索引」和「建表后追加索引」两种，核心语法如下：

1. **建表时创建索引**
    
    sql
    
    ```
    CREATE TABLE `user` (
      `id` INT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
      `user_name` VARCHAR(50) NOT NULL COMMENT '用户名',
      `phone` CHAR(11) NOT NULL COMMENT '手机号',
      `email` VARCHAR(100) NOT NULL COMMENT '邮箱',
      `age` TINYINT NOT NULL COMMENT '年龄',
      `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
      -- 主键索引
      PRIMARY KEY (`id`),
      -- 唯一索引
      UNIQUE INDEX `uk_phone` (`phone`),
      -- 普通单值索引
      INDEX `idx_age` (`age`),
      -- 联合索引（最左前缀原则）
      INDEX `idx_name_email_create_time` (`user_name`, `email`, `create_time`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表';
    ```
    
2. **建表后追加索引**
    
    sql
    
    ```
    -- 普通索引
    CREATE INDEX `idx_age` ON `user` (`age`);
    -- 唯一索引
    CREATE UNIQUE INDEX `uk_email` ON `user` (`email`);
    -- 联合索引
    CREATE INDEX `idx_name_age` ON `user` (`user_name`, `age`);
    -- 主键索引（表只能有一个主键）
    ALTER TABLE `user` ADD PRIMARY KEY (`id`);
    ```
    
3. **删除索引**
    
    sql
    
    ```
    DROP INDEX `idx_age` ON `user`;
    ALTER TABLE `user` DROP PRIMARY KEY;
    ```
    

##### 二、创建索引的核心原则

1. **优先为查询、排序、分组的字段创建索引**：索引仅用于提升查询效率，仅为 WHERE 条件、JOIN 关联字段、ORDER BY、GROUP BY 的字段创建索引，不要为很少查询的字段创建索引。
2. **优先选择区分度高的字段创建索引**：区分度 = 不重复值数量 / 总数据量，区分度越接近 1，索引效率越高。比如手机号、身份证号、邮箱，区分度极高，适合建索引；性别、状态字段，只有 2-3 个值，区分度极低，不适合建索引。
3. **联合索引遵循最左前缀原则**：联合索引的字段顺序，必须把区分度高、查询最频繁的字段放在最左侧，后续字段按区分度从高到低排序，查询时必须匹配最左前缀，索引才会生效。
4. **避免过度建索引**：索引不是越多越好，每个索引都会占用存储空间，同时会降低 INSERT、UPDATE、DELETE 的性能（数据修改时，需要同步更新所有对应的索引），只创建必要的索引。
5. **小表不需要建索引**：数据量很小的表（比如几百行数据），全表扫描的速度比走索引更快，不需要创建索引。
6. **避免在索引字段上做函数运算、类型转换、表达式计算**：这些操作会导致索引失效，创建索引时，要保证查询时能直接使用索引字段，避免函数操作。
7. **优先使用联合索引，避免多个单值索引**：多个查询条件共用一个联合索引，比多个单值索引效率更高，同时能减少索引的数量，降低维护成本。
8. **长字符串字段，优先使用前缀索引**：对于 varchar (255) 这类长字符串字段，比如文章标题、URL，可创建前缀索引，只对前 N 个字符建索引，比如`CREATE INDEX idx_title_prefix ON article (title(20))`，节省索引存储空间，提升查询效率。

#### 专家级拓展

- 索引失效的高频场景（创建索引时必须规避）：
    
    1. 索引字段使用函数、表达式计算、类型转换；
    2. 模糊查询以 % 开头，比如`WHERE name LIKE '%张三'`；
    3. 联合查询不遵循最左前缀原则，跳过了最左侧字段；
    4. 使用 OR 连接非索引字段，比如`WHERE name='张三' OR age=20`，age 没有索引，会导致全表扫描；
    5. 使用！=、<>、NOT IN 等反向查询，大概率会导致索引失效；
    6. 索引字段允许为 NULL，使用 IS NULL/IS NOT NULL 查询，可能导致索引失效。
    
- 生产环境最佳实践：
    
    1. 单表索引数量控制在 5 个以内，联合索引的字段数量不超过 5 个；
    2. 必须为每张表创建主键索引，优先使用自增 INT/BIGINT 类型的主键，避免使用 UUID 作为主键，会导致索引碎片化，性能下降；
    3. 上线前必须用 EXPLAIN 分析 SQL 的执行计划，确认索引是否生效，避免上线后出现全表扫描；
    4. 定期用 pt-index-usage 工具分析索引的使用情况，删除长期未使用的无效索引，节省存储空间，提升写入性能。
    

#### 面试避坑指南

- 严禁创建索引的语法写错，比如字段顺序、关键字错误，会被面试官判定为无实际 SQL 开发经验；
- 避免只说创建语法，不说索引设计的核心原则，面试问这个问题，核心是考察你的索引设计能力，不是语法背诵能力；
- 不要说「索引越多越好」，这是新手常见误区，会被面试官判定为无生产环境实战经验。