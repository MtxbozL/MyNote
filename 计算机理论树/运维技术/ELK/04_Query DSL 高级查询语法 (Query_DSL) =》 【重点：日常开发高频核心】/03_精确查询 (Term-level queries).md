
精确值查询是针对结构化数据的精准匹配方案，专门适配`keyword`、数值、日期、布尔等类型字段，核心特性是：**不对查询文本执行任何分词处理**，直接将输入的完整值作为单个词条，去倒排索引中执行精准匹配，仅当词条完全一致时才会命中。

### 4.3.1 核心特性与顶级避坑指南

#### 核心特性

1. 无分词处理：查询文本原样匹配，不做任何大小写转换、分词拆分；
2. 精准匹配：仅当倒排索引中的词条与查询值完全一致时，才会匹配成功；
3. 无相关性计分：绝大多数精确查询都应放在过滤上下文中，不计分、可缓存，性能最优；
4. 适配字段类型：仅适用于`keyword`、integer、long、date、boolean、ip 等不分词的字段类型。

#### 顶级避坑指南

**严禁对`text`类型字段使用 term/terms 等精确查询**，这是 ES 新手最高发的致命错误。

- 错误原因：`text`类型字段写入时已被分词，拆分为多个子词条，倒排索引中仅存储分词后的子词条，不存在完整的原始文本词条；
- 错误示例：`text`类型字段内容为 “我爱中国”，分词后为`[我,爱,中国]`，使用`term`查询 “我爱中国”，倒排索引中无该完整词条，查询结果为空；
- 正确方案：使用`text`字段的`.keyword`子字段执行精确查询，或使用 match 全文查询。

### 4.3.2 term/terms 查询（单值 / 多值精确匹配）

#### 4.3.2.1 term 查询（单值精确匹配）

term 查询是精确查询的基础，用于匹配字段值与查询值**完全相等**的文档，等价于 SQL 中的`WHERE field = 'value'`。

##### 标准语法与示例

```bash
GET /<索引名称>/_search
{
  "query": {
    "term": {
      "<目标字段名>": {
        "value": "<精确匹配值>",
        "boost": 1.0
      }
    }
  }
}
```

典型生产示例：

```bash
# 1. 精确匹配订单状态为已支付的文档（keyword类型）
GET /order_info/_search
{
  "query": {
    "term": {
      "order_status.keyword": {
        "value": "paid"
      }
    }
  }
}

# 2. 精确匹配用户ID为10001的文档（long类型），放在过滤上下文，性能最优
GET /user_info/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "user_id": 10001 } }
      ]
    }
  }
}
```

#### 4.3.2.2 terms 查询（多值精确匹配）

terms 查询是 term 查询的多值扩展版本，用于匹配字段值属于查询值列表中任意一个的文档，等价于 SQL 中的`WHERE field IN ('value1', 'value2', 'value3')`。

##### 标准语法与示例

```bash
GET /<索引名称>/_search
{
  "query": {
    "terms": {
      "<目标字段名>": ["值1", "值2", "值3", "..."]
    }
  }
}
```

典型生产示例：

```bash
# 匹配订单状态为已支付、已发货、已完成的文档
GET /order_info/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "terms": {
            "order_status.keyword": ["paid", "shipped", "completed"]
          }
        }
      ]
    }
  }
}
```

##### 核心限制与注意事项

- terms 查询的值列表默认最大长度为 65536，可通过`index.max_terms_count`配置调整，生产环境不建议设置过大，避免查询性能下降；
- terms 查询不支持跨字段匹配，仅能针对单个字段做多值匹配；
- 所有值必须与字段类型完全匹配，例如数值字段不能传入字符串值，否则会导致匹配失败。

### 4.3.3 range 查询（范围查询）

range 查询用于匹配字段值在指定范围内的文档，适配数值、日期、ip 等可排序的字段类型，等价于 SQL 中的`WHERE field >= x AND field <= y`，是日志分析、时序数据统计、数值筛选的核心查询类型。

#### 4.3.3.1 核心参数

|参数名|含义|
|---|---|
|`gte`|大于等于（greater than or equal）|
|`gt`|大于（greater than）|
|`lte`|小于等于（less than or equal）|
|`lt`|小于（less than）|
|`format`|日期格式，覆盖字段 mapping 中定义的默认格式|
|`time_zone`|时区，解决日期查询的时区偏移问题，国内推荐`Asia/Shanghai`|

#### 4.3.3.2 典型示例

1. **数值范围查询**
    
    ```bash
    # 匹配订单金额在100~1000元之间的文档
    GET /order_info/_search
    {
      "query": {
        "bool": {
          "filter": [
            {
              "range": {
                "order_amount": {
                  "gte": 100,
                  "lte": 1000
                }
              }
            }
          ]
        }
      }
    }
    ```
    
2. **日期范围查询（生产高频）**
    
    ```bash
    # 匹配最近7天内创建的订单，指定东八区时区
    GET /order_info/_search
    {
      "query": {
        "bool": {
          "filter": [
            {
              "range": {
                "create_time": {
                  "gte": "now-7d/d",
                  "lte": "now/d",
                  "time_zone": "Asia/Shanghai"
                }
              }
            }
          ]
        }
      }
    }
    ```
    
    日期表达式说明：
    
    - `now`：当前系统时间；
    - `now-7d`：当前时间往前推 7 天；
    - `/d`：按天取整，`now-7d/d`表示 7 天前的 00:00:00，`now/d`表示当天的 00:00:00，可大幅提升过滤条件的缓存命中率，是生产环境的最佳实践。
    
3. **IP 地址范围查询**
    
    ```bash
    # 匹配内网IP段的访问日志
    GET /access_log/_search
    {
      "query": {
        "bool": {
          "filter": [
            {
              "range": {
                "client_ip": {
                  "gte": "192.168.0.0",
                  "lte": "192.168.255.255"
                }
              }
            }
          ]
        }
      }
    }
    ```
    

### 4.3.4 exists 查询（字段存在性判断）

exists 查询用于匹配**字段存在且有有效值**的文档，等价于 SQL 中的`WHERE field IS NOT NULL`，对应的字段不存在查询等价于`WHERE field IS NULL`。

#### 4.3.4.1 核心判断规则

ES 对字段 “存在且有效” 的定义有严格规则，是新手高频踩坑点，具体如下：

- 【字段存在】：字段在文档中出现，且值不属于无效值；
- 【字段不存在】：字段未在文档中出现，或值为以下无效值：
    
    1. `null`或`[]`（空数组）；
    2. 数组中所有元素均为`null`；
    
- 【重点避坑】：空字符串`""`、空格字符串`" "`、数组包含空字符串`[""]`、数值 0、布尔值 false，均属于有效值，exists 查询会判定为字段存在。

#### 4.3.4.2 标准语法与示例

1. **字段存在查询（IS NOT NULL）**
    
    ```bash
    # 匹配user_email字段存在且有有效值的文档
    GET /user_info/_search
    {
      "query": {
        "exists": {
          "field": "user_email"
        }
      }
    }
    ```
    
2. **字段不存在查询（IS NULL）**
    
    需通过`bool.must_not` + `exists`组合实现，无单独的`not_exists`查询：
    
    ```bash
    # 匹配user_email字段不存在或为无效值的文档
    GET /user_info/_search
    {
      "query": {
        "bool": {
          "must_not": [
            {
              "exists": {
                "field": "user_email"
              }
            }
          ]
        }
      }
    }
    ```
