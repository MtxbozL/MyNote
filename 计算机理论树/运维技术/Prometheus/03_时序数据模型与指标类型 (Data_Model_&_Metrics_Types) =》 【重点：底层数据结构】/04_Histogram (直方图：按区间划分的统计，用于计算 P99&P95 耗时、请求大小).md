
Histogram 是 Prometheus 用于**观测值分布统计**的核心指标类型，专为解决平均值无法反映长尾分布的痛点设计，支持服务端计算 P99/P95/P90 等分位数指标，是接口性能监控、耗时分布统计的首选方案。

### 3.4.1 核心定义与数据结构

Histogram 是一种**分桶累积型统计指标**，通过预定义的数值区间（桶，Bucket）对观测值进行计数，同时统计观测值的总数与累积总和，完整的 Histogram 指标由 3 组时序构成：

1. `[metric_name]_bucket{le="x"}`：**累积计数器（Counter 类型）**，记录观测值 ≤ x 的事件总次数，`le`为`less than or equal`的缩写，是 Histogram 的核心内置标签；
2. `[metric_name]_sum`：**累积计数器（Counter 类型）**，记录所有观测值的数值总和；
3. `[metric_name]_count`：**累积计数器（Counter 类型）**，记录所有观测值的总次数，与`[metric_name]_bucket{le="+Inf"}`的值完全相等。

> 核心约束：Histogram 必须包含`le="+Inf"`的桶，该桶的计数值为所有观测值的总数，是分位数计算的基础。

#### 示例说明

以接口请求耗时指标`http_request_duration_seconds`为例，预定义桶为`0.1, 0.3, 0.5, 1.0, +Inf`，若 100 次请求的耗时分布为：

- ≤0.1s：20 次
- 0.1s < 耗时 ≤0.3s：30 次
- 0.3s < 耗时 ≤0.5s：30 次
- 0.5s < 耗时 ≤1.0s：15 次
- > 1.0s：5 次
    

则对应的 Histogram 时序值为：

```bash
http_request_duration_seconds_bucket{le="0.1"} 20
http_request_duration_seconds_bucket{le="0.3"} 50
http_request_duration_seconds_bucket{le="0.5"} 80
http_request_duration_seconds_bucket{le="1.0"} 95
http_request_duration_seconds_bucket{le="+Inf"} 100
http_request_duration_seconds_sum 42.5
http_request_duration_seconds_count 100
```

### 3.4.2 核心特性

1. **预定义桶机制**：桶的数值区间必须在客户端埋点时提前定义，Prometheus 服务端无法修改桶配置；
2. **服务端分位数计算**：通过`histogram_quantile()`函数在服务端计算分位数，支持跨实例、跨标签维度的聚合；
3. **存储开销可控**：时序基数由桶的数量决定，与观测值的总次数无关，即使百万级观测值，只要桶数量固定，时序基数保持稳定；
4. **线性插值精度**：分位数计算基于桶的累积计数进行线性插值，结果为近似值，桶的划分越贴合观测值分布，精度越高。

### 3.4.3 核心 PromQL 函数与使用规范

Histogram 的核心适配函数为`histogram_quantile(φ, <range_vector>)`，用于计算分位数：

- 参数`φ`：分位数阈值，取值范围为 0 < φ < 1，例如`0.99`对应 P99 分位数，`0.95`对应 P95 分位数；
- 参数`<range_vector>`：必须是包含`le`标签的 Histogram 桶时序向量。

#### 标准查询示例

```bash
# 计算HTTP请求耗时的P99分位数（5分钟窗口）
histogram_quantile(0.99, sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))

# 计算跨实例聚合的接口P95耗时
histogram_quantile(0.95, sum by (le, api_path) (rate(http_request_duration_seconds_bucket[10m])))
```

#### 核心使用规范

**必须严格遵循「先 rate，后聚合，再计算分位数」的执行顺序**，错误的顺序会导致分位数计算结果完全失真：

- 正确顺序：`rate()` 计算每个桶的增长速率 → `sum()` 按`le`标签聚合 → `histogram_quantile()` 计算分位数；
- 错误顺序：先聚合桶数据，再计算 rate，会破坏 Histogram 的累积计数特性，导致插值计算失效。

### 3.4.4 高频使用误区与禁忌

1. 桶的划分不合理，例如桶的区间无法覆盖大部分观测值，导致绝大多数观测值落入`+Inf`桶，分位数计算完全失效；
2. 桶的数量过多（超过 50 个），导致时序基数爆炸，例如 20 个桶的 Histogram，每添加 1 个标签维度，时序数量会放大 20 倍；
3. 忘记添加`le="+Inf"`桶，导致`_count`与桶总数不一致，`histogram_quantile()`函数无法正常工作；
4. 对 Histogram 的`_sum`与`_count`使用 Gauge 适配函数，二者均为 Counter 类型，必须使用`rate()`/`increase()`函数。

---
