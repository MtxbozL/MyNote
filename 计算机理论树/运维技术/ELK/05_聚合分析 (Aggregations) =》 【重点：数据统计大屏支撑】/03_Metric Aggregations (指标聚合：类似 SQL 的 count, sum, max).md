
指标聚合是 ES 统计计算的核心载体，其核心逻辑是 **对一个文档集（全量查询结果集或分桶后的单个桶内文档集）执行数值计算，输出一个或多个统计指标值** ，等价于 SQL 中的聚合函数，分为单值指标聚合与多值指标聚合两大类。

指标聚合通常与分桶聚合嵌套使用，实现 “分组后统计” 的核心需求，是业务指标计算的核心基础。本节聚焦目录指定的核心指标聚合类型，覆盖全场景数值统计需求。

### 5.3.1 单值指标聚合

单值指标聚合输出单个数值结果，是最基础、最常用的指标类型，与 SQL 聚合函数完全对应，核心类型如下：

|聚合类型|核心定义|等价 SQL|适用场景|
|---|---|---|---|
|`avg`|计算字段的平均值|`AVG(field)`|平均响应时间、平均客单价、平均销量统计|
|`max`|计算字段的最大值|`MAX(field)`|峰值响应时间、最高销售额、最大并发量统计|
|`min`|计算字段的最小值|`MIN(field)`|最低延迟、最低价格、最小库存统计|
|`sum`|计算字段的总和|`SUM(field)`|总销售额、总订单量、总访问量统计|
|`value_count`|统计字段有非空值的文档数量|`COUNT(field)`|有效请求数、有值的字段数量统计|

#### 标准语法与通用示例

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
    "total_order_count": { "value_count": { "field": "order_id.keyword" } },
    "total_sales": { "sum": { "field": "order_amount" } },
    "avg_order_amount": { "avg": { "field": "order_amount" } },
    "max_order_amount": { "max": { "field": "order_amount" } },
    "min_order_amount": { "min": { "field": "order_amount" } }
  }
}
```

该示例一次性计算出最近 30 天已支付订单的总订单数、总销售额、平均客单价、最高订单金额、最低订单金额，无需多次查询，性能极高。

### 5.3.2 多值指标聚合

多值指标聚合输出多个统计数值，一次性获取字段的全量统计信息，避免多次聚合查询的性能开销，核心类型为`stats`与`extended_stats`。

#### 5.3.2.1 Stats Aggregation（基础统计聚合）

`stats`聚合一次性输出字段的 6 个核心统计指标，包含：`count`（文档数）、`min`（最小值）、`max`（最大值）、`avg`（平均值）、`sum`（总和），等价于同时执行 5 个单值指标聚合，是数据探索性分析的首选。

标准语法示例：

```bash
GET /access_log/_search
{
  "size": 0,
  "aggs": {
    "response_time_stats": {
      "stats": {
        "field": "response_time_ms"
      }
    }
  }
}
```

#### 5.3.2.2 Extended Stats Aggregation（扩展统计聚合）

`extended_stats`聚合是`stats`的扩展版本，在基础统计指标之上，新增了方差、标准差、平方和、均值偏差等统计学指标，适用于专业的数据分析、异常检测场景。

### 5.3.3 Cardinality Aggregation（基数去重统计）

Cardinality 聚合是 ES 实现去重统计的核心方案，**用于统计字段的唯一值数量，等价于 SQL 中的`COUNT(DISTINCT field)`语句**，是 UV 统计、独立用户数、独立 IP 数等场景的核心聚合类型。

#### 5.3.3.1 核心原理：HyperLogLog++ 近似算法

ES 是分布式架构，若要实现 100% 精确的去重统计，需要将所有分片的全量唯一值拉取到协调节点，在内存中做全局去重。当唯一值数量达到千万级以上时，会占用大量内存，导致集群 OOM，查询性能急剧下降。

为解决该问题，ES 的 Cardinality 聚合采用**HyperLogLog++（HLL++）** 概率近似算法，核心特性如下：

1. **内存占用固定**：无论唯一值数量多少，单个 Cardinality 聚合最多占用约 48KB 内存，内存开销完全可控；
2. **低延迟计算**：算法基于哈希值的前缀统计，无需存储全量唯一值，计算速度极快，适配实时统计场景；
3. **误差率可控**：默认误差率约为 0.5%，可通过参数调整精度，完全满足监控、统计、大屏等绝大多数业务场景的需求。

#### 5.3.3.2 核心语法与参数

```bash
"aggs": {
  "<自定义聚合名称>": {
    "cardinality": {
      "field": "<去重字段名>",  // 必选参数，keyword、数值、date类型
      "precision_threshold": 3000 // 可选，精度阈值，范围100-40000，默认3000
    }
  }
}
```

核心参数`precision_threshold`详解：

- 该参数定义了精度与内存的平衡点，阈值内的唯一值数量，误差率极低，接近精确值；
- 阈值越大，精度越高，内存占用越大，内存占用与阈值呈线性增长关系；
- 默认值 3000，在该阈值下，误差率可控制在 0.5% 以内，平衡精度与内存开销；
- 最大值 40000，超过该值后，精度不会再提升，内存占用持续增长，生产环境不建议设置超过 40000。

#### 5.3.3.3 生产级示例与避坑指南

以网站访问日志场景为例，按天统计独立访客 UV（独立 IP 数）：

```bash
GET /access_log/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "range": { "request_time": { "gte": "now-30d/d", "time_zone": "Asia/Shanghai" } } }
      ]
    }
  },
  "aggs": {
    "daily_group": {
      "date_histogram": {
        "field": "request_time",
        "calendar_interval": "day",
        "time_zone": "Asia/Shanghai",
        "format": "yyyy-MM-dd"
      },
      "aggs": {
        "uv_count": {
          "cardinality": {
            "field": "client_ip.keyword",
            "precision_threshold": 10000
          }
        }
      }
    }
  }
}
```

#### 核心避坑指南

1. **精度限制**：Cardinality 聚合是近似算法，无法实现 100% 精确去重，严禁用于金融对账、交易统计等需要绝对精确的场景；
2. **字段限制**：仅支持开启 Doc Values 的 keyword、数值、date 类型，严禁对 text 类型字段执行去重统计；
3. **嵌套场景优化**：在分桶聚合内嵌套 Cardinality 聚合时，需控制分桶数量，避免大量分桶导致的内存开销累加；
4. **超大数据量优化**：亿级以上数据的去重统计，建议通过预聚合、离线计算实现，避免实时聚合的性能损耗。
