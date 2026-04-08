
PromQL 支持对两个瞬时向量进行二元运算，运算的核心是**向量匹配**，即基于标签集找到两个向量中可配对的时序，完成运算。同时支持集合运算，实现时序数据的交集、并集、补集过滤。

### 4.5.1 向量匹配核心逻辑

PromQL 二元运算的默认匹配规则：

1. 遍历左侧向量的每一条时序，在右侧向量中寻找**标签集完全相等**的时序；
2. 若找到匹配的时序对，执行二元运算，生成新的时序，标签集保留匹配的标签；
3. 若未找到匹配的时序，该时序会被丢弃，不会出现在结果中。

向量匹配支持两个修饰符，用于自定义匹配规则：

- `on(<label_list>)`：仅使用指定的标签列表进行匹配，忽略其余标签；
- `ignoring(<label_list>)`：忽略指定的标签列表，使用剩余的标签进行匹配。

### 4.5.2 一对一向量匹配

一对一匹配是最基础的向量匹配模式，左右两个向量中的时序一一对应，每条时序仅能匹配到对方的一条时序，适用于相同维度指标的关联计算。

#### 标准示例

```bash
# 基础一对一匹配：计算HTTP请求错误率
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])

# 带ignoring修饰符的匹配：忽略status标签，计算相同job、instance、path的错误率
rate(http_requests_total{status=~"5.."}[5m]) / ignoring(status) rate(http_requests_total[5m])

# 带on修饰符的匹配：仅用job和instance标签匹配，计算实例CPU使用率与内存使用率的比值
avg by (job, instance) (rate(node_cpu_seconds_total[1m])) / on(job, instance) avg by (job, instance) (node_memory_usage_ratio)
```

#### 核心适用场景

- 相同维度指标的比值计算，如错误率、成功率、使用率；
- 相同标签集的指标关联运算，如加减乘除、比较运算；
- 告警规则中的多条件组合判断。

### 4.5.3 一对多 / 多对一向量匹配（`group_left`/`group_right`）

一对多 / 多对一匹配是高级向量匹配模式，适用于左右两个向量的时序数量不对等的场景：

- 一对多匹配：左侧向量的一条时序，可匹配右侧向量的多条时序，使用`group_left`修饰符；
- 多对一匹配：右侧向量的一条时序，可匹配左侧向量的多条时序，使用`group_right`修饰符。

该模式是多维度指标与聚合指标关联计算的核心实现方式，必须与`on()`/`ignoring()`修饰符组合使用。

#### 语法规范

```java
<left_vector> <operator> on(<label_list>) group_left(<label_list>) <right_vector>

<left_vector> <operator> on(<label_list>) group_right(<label_list>) <right_vector>
```

- `group_left`/`group_right`后的标签列表为可选参数，用于指定从对方向量中继承的额外标签；
- 若未指定标签列表，默认继承对方向量中`on()`未指定的所有标签。

#### 标准示例

```bash
# 一对多匹配：计算每个实例的CPU使用率与所在集群的平均CPU使用率的比值
# 左侧：每个实例的CPU使用率（多时序，标签：cluster, instance）
# 右侧：每个集群的平均CPU使用率（少时序，标签：cluster）
avg by (cluster, instance) (node_cpu_usage_ratio)
/ on(cluster) group_left()
avg by (cluster) (node_cpu_usage_ratio)

# 多对一匹配：计算每个接口的耗时与该服务全局平均耗时的差值
# 左侧：每个接口的平均耗时（多时序，标签：service, api_path）
# 右侧：每个服务的全局平均耗时（少时序，标签：service）
avg by (service, api_path) (http_request_duration_seconds)
- on(service) group_right()
avg by (service) (http_request_duration_seconds)

# 带标签继承的匹配：计算每个实例的请求量占比，继承服务的SLA阈值标签
rate(http_requests_total[5m])
/ on(job) group_left(sla_threshold)
sum by (job) (rate(http_requests_total[5m]))
```

#### 核心适用场景

- 多维度明细指标与聚合指标的关联计算，如占比、差值、倍数；
- 单条全局规则与多条明细时序的匹配，如阈值对比、基准值对比；
- 多标签指标与少标签指标的关联运算，如实例指标与集群指标的匹配。

### 4.5.4 集合运算

PromQL 支持三类集合运算，用于对两个瞬时向量的时序标识进行集合操作，**仅匹配时序标识，不关注数值**，运算结果保留左侧向量的数值与标签集。

|集合运算符|语义|核心逻辑|
|---|---|---|
|`and`|交集运算|仅保留**左右两个向量中同时存在**的时序（标签集完全匹配）|
|`or`|并集运算|保留左侧向量的所有时序，以及右侧向量中**左侧不存在**的时序|
|`unless`|补集运算|仅保留**左侧向量中存在、右侧向量中不存在**的时序（标签集完全匹配）|

#### 标准示例与适用场景

```bash
# and交集运算：筛选CPU使用率>80% 且 内存使用率>80%的实例
node_cpu_usage_ratio > 80 and node_memory_usage_ratio > 80

# and过滤：仅保留处于运行状态的实例的高磁盘使用率指标
node_filesystem_usage_ratio > 90 and up{job="node_exporter"} == 1

# or并集运算：主指标不存在时，使用备用指标兜底
primary_service_requests_total or fallback_service_requests_total

# unless补集运算：排除维护中的实例，仅保留非维护状态的离线实例
up{job="node_exporter"} == 0 unless maintenance_mode == 1

# unless过滤：排除测试环境的告警指标，仅保留生产环境的异常指标
http_error_rate > 0.05 unless env=~"test|dev"
```

---

## 4.6 PromQL 运算符优先级

PromQL 二元运算符的优先级从高到低排序如下，优先级相同的运算符按从左到右的顺序执行，可通过括号`()`改变执行顺序：

1. `^`（幂运算，右结合）
2. `*`、`/`、`%`（乘、除、取模）
3. `+`、`-`（加、减）
4. `==`、`!=`、`<=`、`<`、`>=`、`>`（比较运算）
5. `and`、`unless`（集合运算）
6. `or`（并集运算）

标准示例：

```bash
# 优先级示例：先乘除后加减，括号优先
(rate(http_requests_total[5m]) * 100) / (rate(http_requests_total[5m]) + rate(http_errors_total[5m]))

# 幂运算示例：计算磁盘使用率的平方
(node_filesystem_used_bytes / node_filesystem_size_bytes) ^ 2
```

---

## 4.7 高频使用误区与生产环境最佳实践

### 4.7.1 高频使用误区

1. **指标类型误用**：对 Gauge 使用`rate()`，对 Counter 使用`deriv()`/`predict_linear()`，导致计算结果完全错误；
2. **聚合顺序错误**：先聚合 Counter 原始值，再计算速率，破坏了 Counter 的累积特性，导致速率计算失真；
3. **Histogram 分位数计算顺序错误**：先聚合再计算 rate，导致分位数插值失效；
4. **窗口大小不合理**：使用小于 2 倍抓取间隔的窗口，导致速率计算无数据或结果失真；
5. **向量匹配标签不匹配**：未使用`on()`/`ignoring()`修饰符，导致标签集不匹配，查询无结果；
6. **高基数查询**：对高基数标签进行过滤、聚合，导致 Prometheus 内存占用激增，查询超时。

### 4.7.2 生产环境最佳实践

1. **查询性能优化**：优先使用相等匹配，减少正则匹配；避免全量指标查询，通过标签过滤缩小查询范围；避免大时间窗口的高基数查询；
2. **告警规则规范**：告警表达式优先使用`rate()`，禁止使用`irate()`；窗口大小至少覆盖 3 个抓取间隔，避免瞬时波动触发误告警；
3. **预聚合优化**：对高频查询的复杂表达式，通过记录规则（Recording Rule）进行预聚合，提升查询性能；
4. **语法校验**：使用`promtool`工具校验 PromQL 表达式的合法性，避免语法错误；
5. **标签规范**：严格控制标签基数，禁止在标签中加入高基数的唯一标识，从源头避免高基数问题。

---

## 本章小结

本章完成了 PromQL 查询语言的系统性学习，核心内容包括：

1. 建立了 PromQL 的核心认知，明确了其声明式多维查询的设计理念，深入理解了瞬时向量与区间向量的数学定义、本质区别与适用场景；
2. 熟练掌握了 4 类标签匹配器的语义、语法与性能规范，实现了多维度数据的精准过滤，理解了指标名称的底层标签实现；
3. 掌握了`offset`时间位移与`@`绝对时间修饰符的用法，实现了环比、同比、定点时间的标准化查询；
4. 深度拆解了四大类核心函数的实现原理与数学逻辑，重点掌握了速率计算函数的选型规范、聚合函数的分组规则、趋势预测函数的适用场景、分位数计算函数的执行顺序；
5. 理解了向量匹配的核心逻辑，掌握了一对一、一对多匹配的实现方式，解决了多维度指标的关联计算问题；
6. 掌握了三类集合运算的语义与适用场景，实现了时序数据的交集、并集、补集过滤；
7. 明确了 PromQL 的运算符优先级、高频使用误区与生产环境最佳实践，建立了标准化的 PromQL 编写规范。

本章内容是 Prometheus 日常运维、告警配置、故障排查的核心基础，也是面试考察的重中之重，为后续 Exporters 生态、服务发现、告警配置等内容的学习提供了核心查询能力支撑。