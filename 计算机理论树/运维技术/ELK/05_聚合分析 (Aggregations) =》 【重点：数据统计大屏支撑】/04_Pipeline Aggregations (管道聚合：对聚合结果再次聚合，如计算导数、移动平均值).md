
管道聚合是 ES 实现复杂趋势分析的核心能力，其核心逻辑是 **聚合的输入不是原始文档集，而是其他聚合的输出结果，在已有聚合结果的基础上执行二次计算** ，实现传统 SQL 难以实现的环比增长率、移动平均值、累计和、峰值检测等复杂分析需求。

管道聚合的核心语法规则是通过`buckets_path`参数指定要引用的聚合结果路径，路径格式为`<父聚合名称>.<子聚合名称>`，支持多级嵌套路径引用。根据聚合的位置与引用关系，管道聚合分为两大类：兄弟管道聚合与父管道聚合。

### 5.4.1 管道聚合的核心分类

|分类|定义|执行位置|核心适用场景|
|---|---|---|---|
|兄弟管道聚合（Sibling Pipeline）|与原聚合同级，对原聚合所有桶的指标值进行全局计算|与被引用的聚合同级，位于同一 aggs 层级|找出所有时间桶中的销售额峰值、计算所有品牌的平均销售额、全局统计值计算|
|父管道聚合（Parent Pipeline）|嵌套在父分桶聚合内，对父聚合的每个桶的历史数据进行序列计算|嵌套在被引用的分桶聚合内，位于其子 aggs 层级|计算环比增长率、7 天移动平均值、累计销售额、时序数据趋势分析|

### 5.4.2 核心兄弟管道聚合

兄弟管道聚合与原分桶聚合同级，对所有桶的指标值执行全局统计，核心常用类型如下：

#### 5.4.2.1 Max/Min/Sum/Avg Bucket Aggregation

用于找出所有桶中指标值的最大值 / 最小值 / 总和 / 平均值，同时返回对应的桶 key，适用于峰值 / 谷值检测、全局统计场景。

标准示例：找出最近 30 天中，日销售额最高的日期与对应销售额

```bash
GET /order_info/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "order_status.keyword": "paid" } },
        { "range": { "pay_time": { "gte": "now-30d/d", "time_zone": "Asia/Shanghai" } } }
      ]
    }
  },
  "aggs": {
    "daily_sales": {
      "date_histogram": {
        "field": "pay_time",
        "calendar_interval": "day",
        "time_zone": "Asia/Shanghai"
      },
      "aggs": {
        "sales_amount": { "sum": { "field": "order_amount" } }
      }
    },
    "max_daily_sales": {
      "max_bucket": {
        "buckets_path": "daily_sales>sales_amount"
      }
    }
  }
}
```

其中`buckets_path: "daily_sales>sales_amount"`表示引用`daily_sales`分桶聚合内的`sales_amount`指标聚合结果，`>`符号表示聚合层级的嵌套关系。

#### 5.4.2.2 Stats Bucket Aggregation

对所有桶的指标值执行全量统计，输出 count、min、max、avg、sum 等指标，一次性获取所有桶的统计分布情况。

### 5.4.3 核心父管道聚合

父管道聚合嵌套在分桶聚合内，对时序桶的序列数据执行趋势计算，是目录指定的核心重点，覆盖导数、移动平均值等核心场景。

#### 5.4.3.1 Derivative Aggregation（导数聚合）

导数聚合用于计算时序桶中指标值的**一阶 / 二阶导数**，也就是相邻桶之间的数值变化率，核心用于计算环比增长率、增长趋势，是业务趋势分析的核心工具。

标准示例：计算日销售额的环比增长率

```bash
GET /order_info/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "order_status.keyword": "paid" } },
        { "range": { "pay_time": { "gte": "now-30d/d", "time_zone": "Asia/Shanghai" } } }
      ]
    }
  },
  "aggs": {
    "daily_sales": {
      "date_histogram": {
        "field": "pay_time",
        "calendar_interval": "day",
        "time_zone": "Asia/Shanghai",
        "min_doc_count": 0
      },
      "aggs": {
        "sales_amount": { "sum": { "field": "order_amount" } },
        "sales_derivative": {
          "derivative": {
            "buckets_path": "sales_amount",
            "unit": "day"
          }
        },
        "month_on_month_rate": {
          "bucket_script": {
            "buckets_path": {
              "current": "sales_amount.value",
              "prev": "sales_derivative.value"
            },
            "script": "params.prev == null || params.prev == 0 ? 0 : (params.current - params.prev) / params.prev * 100"
          }
        }
      }
    }
  }
}
```

该示例实现了：

1. 按天分桶计算每日销售额；
2. 通过导数聚合计算相邻两天的销售额差值；
3. 通过桶脚本计算日环比增长率，是业务报表的核心指标。

#### 5.4.3.2 Moving Average Aggregation（移动平均值聚合）

移动平均值聚合用于计算时序桶中，指定滑动窗口内的指标值的平均值，核心作用是平滑时序数据的随机波动，消除数据毛刺，凸显长期趋势，是监控大盘、趋势预测的核心工具。

ES 提供多种移动平均模型，包括简单移动平均（SMA）、线性加权移动平均（LWMA）、指数加权移动平均（EWMA）、霍尔特线性趋势模型（Holt），适配不同的时序数据特性。

标准示例：计算 7 天移动平均销售额，平滑日销售额波动

```bash
GET /order_info/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "order_status.keyword": "paid" } },
        { "range": { "pay_time": { "gte": "now-90d/d", "time_zone": "Asia/Shanghai" } } }
      ]
    }
  },
  "aggs": {
    "daily_sales": {
      "date_histogram": {
        "field": "pay_time",
        "calendar_interval": "day",
        "time_zone": "Asia/Shanghai",
        "min_doc_count": 0
      },
      "aggs": {
        "sales_amount": { "sum": { "field": "order_amount" } },
        "7d_moving_avg": {
          "moving_avg": {
            "buckets_path": "sales_amount",
            "window": 7,
            "model": "simple"
          }
        }
      }
    }
  }
}
```

核心参数详解：

- `window`：滑动窗口大小，默认 5，表示取当前桶及之前的 N 个桶计算平均值，示例中 7 表示 7 天滑动窗口；
- `model`：移动平均模型，`simple`为简单移动平均，对窗口内所有数据赋予相同权重，是最常用的模型。

#### 5.4.3.3 Cumulative Sum Aggregation（累计和聚合）

累计和聚合用于计算时序桶中指标值的累计总和，从第一个桶开始，依次累加每个桶的指标值，适用于累计销售额、累计用户数、累计订单量等场景。

标准示例：计算年度累计销售额

```bash
GET /order_info/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "order_status.keyword": "paid" } },
        { "range": { "pay_time": { "gte": "2026-01-01", "lte": "2026-12-31", "time_zone": "Asia/Shanghai" } } }
      ]
    }
  },
  "aggs": {
    "monthly_sales": {
      "date_histogram": {
        "field": "pay_time",
        "calendar_interval": "month",
        "time_zone": "Asia/Shanghai"
      },
      "aggs": {
        "monthly_amount": { "sum": { "field": "order_amount" } },
        "cumulative_sales": {
          "cumulative_sum": {
            "buckets_path": "monthly_amount"
          }
        }
      }
    }
  }
}
```

### 5.4.4 管道聚合核心避坑指南

1. **`buckets_path`路径规则**：必须严格匹配聚合的嵌套层级，使用`>`符号分隔父聚合与子聚合，引用的聚合必须是同层级的有效聚合，否则会出现路径解析错误；
2. **时序桶连续性**：导数、移动平均等序列计算，要求分桶必须是连续的，需设置`min_doc_count: 0`，避免空桶缺失导致的计算错误；
3. **嵌套限制**：管道聚合只能引用同级或父级的聚合结果，无法引用子级聚合；管道聚合的输出结果，可被其他管道聚合引用，但嵌套层级不宜过深；
4. **排序影响**：对分桶结果执行自定义排序后，会破坏时序桶的顺序，导致序列计算结果错误，时序分析场景严禁对时间桶执行非时间排序。

## 5.5 聚合分析生产级最佳实践

1. **聚合范围最小化原则**：所有聚合查询必须先通过 Query+Filter 缩小目标数据集，严禁对全索引执行聚合计算，过滤后的数据集越小，聚合性能越高；
2. **字段选型强制规范**：聚合字段必须使用开启 Doc Values 的 keyword、数值、date 类型，严禁对 text 类型开启 fielddata 执行聚合，否则会导致 JVM 堆内存溢出，集群宕机；
3. **分桶数量严格控制**：避免生成过多的分桶，terms 聚合的`size`不建议超过 1000，date_histogram 严禁在超大时间范围内使用极小间隔（如 1 年数据按 1 分钟分桶，会生成 50 万 + 个桶），否则会导致协调节点内存溢出；
4. **嵌套层级优化**：聚合嵌套层级建议不超过 3 层，过深的嵌套会导致分片聚合结果过大，协调节点合并计算压力剧增，性能急剧下降；
5. **近似算法合理使用**：去重统计优先使用 Cardinality 聚合，无需追求 100% 精确值，大数据量下精确去重的性能开销是近似算法的 100 倍以上；
6. **预聚合性能优化**：超大数据量、高频访问的可视化大屏，优先使用 ES Rollup 索引做预聚合，将小时级 / 天级的统计结果提前预计算，避免每次查询都执行全量聚合，查询性能可提升 100 倍以上；
7. **关闭不必要的返回结果**：聚合查询必须设置`size: 0`，无需总命中数时设置`track_total_hits: false`，大幅减少网络开销与计算成本。

## 本章小结

本章全面覆盖了 ES 聚合分析的全量核心知识点，核心掌握内容如下：

1. 深入理解了 ES 聚合分析的分布式执行逻辑、标准语法结构与三大核心聚合类型的定位，建立了 “先过滤、后聚合、再二次计算” 的完整分析链路认知；
2. 精通了三大核心分桶聚合的原理与用法，包括 terms 字段值分桶的分布式精度优化、date_histogram 时间分桶的时区与连续性控制、range 自定义范围分桶的区间规则；
3. 精通了指标聚合的全量核心类型，包括基础的单值统计、多值全量统计，重点掌握了 Cardinality 基数去重的 HLL++ 算法原理、精度控制与适用场景；
4. 完全掌握了管道聚合的核心分类与用法，理解了`buckets_path`的路径规则，精通了导数聚合、移动平均值聚合、累计和聚合等二次计算方案，实现了复杂的时序趋势分析；
5. 建立了聚合分析的性能优化思维，掌握了生产环境的核心最佳实践与避坑规则，具备了海量数据下的高性能多维度统计分析能力。