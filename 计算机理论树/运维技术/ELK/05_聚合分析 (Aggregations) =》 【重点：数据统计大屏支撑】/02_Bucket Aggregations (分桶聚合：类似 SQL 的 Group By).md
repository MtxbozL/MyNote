
分桶聚合是 ES 多维度分析的核心基础，其核心逻辑是 **按照预定义的规则，对目标文档集进行分组，每个分组对应一个 “桶（Bucket）”，文档满足该桶的规则则被分配进入桶中** ，最终输出一组桶的集合，每个桶包含对应的文档数量与匹配的文档集。

分桶聚合的核心价值是实现数据的维度拆分，与 SQL 中的`GROUP BY`语句语义等价，但支持无限层级的嵌套分组，能力远超传统 SQL 的分组能力。本节聚焦目录指定的三大核心分桶聚合类型，覆盖 99% 的生产分桶场景。

### 5.2.1 Terms Aggregation（字段值分桶）

Terms 分桶是最常用、最核心的分桶聚合类型，**按照指定字段的唯一值（Term）进行分桶，每个唯一值对应一个独立的桶**，等价于 SQL 中的`GROUP BY field`语句。

#### 5.2.1.1 核心语法与参数

```bash
"aggs": {
  "<自定义聚合名称>": {
    "terms": {
      "field": "<分桶字段名>",  // 必选参数，必须为keyword、数值、date类型
      "size": 10,               // 可选，返回的桶的数量，默认10
      "shard_size": 100,        // 可选，每个分片提前取的Top N桶数量，解决分布式精度问题
      "order": {                // 可选，桶的排序规则
        "_count": "desc"        // 默认按文档数量降序排列
      },
      "min_doc_count": 1,       // 可选，桶的最小文档数，默认1，过滤掉文档数不足的桶
      "include": "<正则表达式>", // 可选，仅包含key匹配正则的桶
      "exclude": "<正则表达式>"  // 可选，排除key匹配正则的桶
    }
  }
}
```

核心参数详解：

1. `field`：分桶字段必须为 **开启 Doc Values 的非 text 类型** ，严禁直接对 text 类型字段执行 terms 分桶。若需对文本内容分桶，必须使用该字段的`.keyword`子字段；
2. `size`：控制最终返回的 Top N 桶的数量，默认值为 10，最大值可通过`index.max_terms_count`配置调整，生产环境不建议设置超过 1000，避免内存溢出；
3. `shard_size`：解决分布式聚合的精度问题的核心参数。ES 是分布式架构，terms 聚合的执行流程为：每个分片取 Top `shard_size`个桶，协调节点合并所有分片的结果，再取 Top `size`个桶返回。默认`shard_size = size * 1.5 + 10`，当`size`较小时，会出现统计结果偏差（如全局 Top 10 的桶，在单个分片内未进入 Top `shard_size`，导致最终结果遗漏）。生产环境中，对精度要求高的场景，需将`shard_size`设置为`size`的 5~10 倍，平衡精度与性能；
4. `order`：支持多种排序规则：
    
    - `_count`：按桶内文档数量排序，默认值；
    - `_key`：按桶的 key 值（字段唯一值）排序；
    - 子聚合指标值：按嵌套的指标聚合结果排序，实现按统计值排序。
    
5. `min_doc_count`：默认值为 1，仅返回文档数≥该值的桶，过滤掉低频无效数据，提升聚合性能；设置为 0 时，会返回无匹配文档的空桶。

#### 5.2.1.2 生产级示例

以电商订单场景为例，实现按品牌分组统计订单数量，并按订单金额排序：

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
    "brand_group": {
      "terms": {
        "field": "brand.keyword",
        "size": 10,
        "shard_size": 100,
        "min_doc_count": 10
      },
      "aggs": {
        "total_sales": {
          "sum": { "field": "order_amount" }
        },
        "order_sort": {
          "bucket_sort": {
            "sort": [ { "total_sales": { "order": "desc" } } ]
          }
        }
      }
    }
  }
}
```

该示例实现了：

1. 先过滤出最近 30 天已支付的订单，作为聚合数据集；
2. 按品牌分桶，过滤掉订单数不足 10 的品牌；
3. 嵌套指标聚合，计算每个品牌的总销售额；
4. 按总销售额降序排序，返回 Top 10 品牌。

### 5.2.2 Date Histogram Aggregation（时间直方图分桶）

Date Histogram 分桶是时序数据分析、监控大盘的核心聚合类型，**按照指定的固定时间间隔，对 date 类型字段进行分桶，每个时间区间对应一个独立的桶**，实现时间维度的趋势统计。

#### 5.2.2.1 核心语法与参数

```bash
"aggs": {
  "<自定义聚合名称>": {
    "date_histogram": {
      "field": "<日期字段名>",  // 必选参数，必须为date类型
      "calendar_interval": "day", // 日历间隔，与fixed_interval二选一
      "fixed_interval": "1h",     // 固定间隔，与calendar_interval二选一
      "time_zone": "Asia/Shanghai", // 必选，国内场景指定东八区，避免时区偏移
      "format": "yyyy-MM-dd HH:mm:ss", // 可选，返回的时间key的格式化字符串
      "min_doc_count": 0,        // 可选，默认0，无数据的时间桶也返回，保证时序连续
      "extended_bounds": {       // 可选，扩展时间边界，强制返回指定区间的所有桶
        "min": "2026-01-01 00:00:00",
        "max": "2026-01-31 23:59:59"
      },
      "offset": "+8h"            // 可选，时间桶的偏移量，调整桶的起始时间
    }
  }
}
```

核心参数详解：

1. **时间间隔核心参数（二选一，生产环境优先使用 calendar_interval）**
    
    - `calendar_interval`：日历感知间隔，按照自然日历规则划分时间区间，支持的取值：`year`、`quarter`、`month`、`week`、`day`、`hour`、`minute`、`second`。其核心优势是适配日历规则，例如`month`间隔会自动适配 28/30/31 天的自然月，`year`间隔会自动适配闰年，是业务统计场景的首选；
    - `fixed_interval`：固定毫秒数间隔，不感知日历规则，严格按照固定时长划分区间，格式为`数字+单位`，支持的单位：`ms`（毫秒）、`s`（秒）、`m`（分钟）、`h`（小时）、`d`（天）。例如`30m`表示固定 30 分钟，`1d`表示固定 24 小时，适用于监控指标的固定周期统计，严禁用于自然月、自然年的业务统计。
    
2. `time_zone`：国内生产环境必须显式指定`Asia/Shanghai`，ES 默认使用 UTC 时区，若不指定会导致时间桶偏移 8 小时，出现统计日期错误的高频问题；
3. `min_doc_count`：默认值为 0，即使时间区间内无匹配文档，也会返回空桶，保证时序数据的连续性，避免折线图、柱状图出现时间断点，是可视化场景的强制规范；
4. `extended_bounds`：强制扩展时间桶的边界，即使边界内无数据，也会返回完整的时间桶，适用于固定时间范围的统计报表，保证所有报表的时间区间完全一致。

#### 5.2.2.2 生产级示例

以接口访问日志场景为例，按每小时分桶统计接口访问量、平均响应时间、错误率：

```bash
GET /access_log/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "range": { "request_time": { "gte": "now-7d/d", "lte": "now/d", "time_zone": "Asia/Shanghai" } } }
      ]
    }
  },
  "aggs": {
    "hourly_trend": {
      "date_histogram": {
        "field": "request_time",
        "calendar_interval": "hour",
        "time_zone": "Asia/Shanghai",
        "format": "yyyy-MM-dd HH:mm:ss",
        "min_doc_count": 0
      },
      "aggs": {
        "total_request": { "value_count": { "field": "request_id.keyword" } },
        "avg_response_time": { "avg": { "field": "response_time_ms" } },
        "error_count": {
          "filter": { "range": { "http_status": { "gte": 400 } } }
        },
        "error_rate": {
          "bucket_script": {
            "buckets_path": {
              "total": "total_request.value",
              "error": "error_count._count"
            },
            "script": "params.total == 0 ? 0 : params.error / params.total * 100"
          }
        }
      }
    }
  }
}
```

该示例实现了：

1. 过滤最近 7 天的访问日志，按每小时分桶，保证时间序列连续；
2. 嵌套指标聚合，统计每小时的总请求数、平均响应时间、错误请求数；
3. 通过桶脚本计算每小时的接口错误率，是监控大盘的核心指标。

### 5.2.3 Range Aggregation（范围分桶）

Range 分桶是自定义区间的分桶聚合类型，**按照用户自定义的数值 / 日期范围进行分桶，每个自定义区间对应一个独立的桶**，适用于非固定间隔的范围统计场景，区别于 date_histogram 的固定间隔自动分桶。

#### 5.2.3.1 核心语法与参数

```bash
"aggs": {
  "<自定义聚合名称>": {
    "range": {
      "field": "<分桶字段名>",  // 必选参数，数值、date类型
      "ranges": [               // 必选参数，自定义区间数组，左闭右开规则
        { "from": 0, "to": 100, "key": "0-100元" },
        { "from": 100, "to": 500, "key": "100-500元" },
        { "from": 500, "key": "500元以上" }
      ],
      "keyed": false            // 可选，默认false，返回数组格式；true返回对象格式
    }
  }
}
```

核心规则与参数详解：

1. **区间规则**：严格遵循**左闭右开**原则，`from`包含边界值，`to`不包含边界值。例如`from:100, to:500`的区间，包含 100，不包含 500，区间之间无缝衔接，无重叠、无遗漏；
2. `ranges`：自定义区间数组，支持仅设置`from`（无上界）或仅设置`to`（无下界），每个区间可通过`key`自定义名称，提升结果可读性；
3. `keyed`：设置为`true`时，返回结果以对象格式呈现，key 为自定义的区间名称，便于前端解析，适配可视化场景。

针对 date 类型字段，ES 提供专用的`date_range`分桶聚合，语法与 range 完全一致，支持日期表达式（如`now-7d`），适配自定义日期区间的统计场景。

#### 5.2.3.2 生产级示例

以电商商品场景为例，按价格区间分桶统计商品数量与平均销量：

```bash
GET /goods_info/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "status.keyword": "on_sale" } }
      ]
    }
  },
  "aggs": {
    "price_range_group": {
      "range": {
        "field": "price",
        "keyed": true,
        "ranges": [
          { "from": 0, "to": 100, "key": "0-100元" },
          { "from": 100, "to": 500, "key": "100-500元" },
          { "from": 500, "to": 1000, "key": "500-1000元" },
          { "from": 1000, "to": 5000, "key": "1000-5000元" },
          { "from": 5000, "key": "5000元以上" }
        ]
      },
      "aggs": {
        "goods_count": { "value_count": { "field": "goods_id" } },
        "avg_sales": { "avg": { "field": "month_sales" } }
      }
    }
  }
}
```

该示例实现了：

1. 过滤出上架状态的商品，按自定义价格区间分桶；
2. 嵌套指标聚合，统计每个价格区间的商品数量、月均销量；
3. 设置`keyed: true`，返回对象格式的结果，便于前端渲染饼图、柱状图。
