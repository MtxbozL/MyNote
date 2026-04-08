
Relabeling 是 Prometheus 内置的、基于标签的动态数据处理流水线，是 Prometheus 高级运维的核心能力，也是实现监控自动化、精细化、低基数的关键手段。其核心作用是在指标的生命周期中，对标签进行修改、添加、删除、过滤，实现抓取目标的筛选、元数据的提取、指标的裁剪。

### 6.4.1 核心生命周期与执行流程

Prometheus 的指标处理流水线分为两个核心 Relabeling 阶段，严格按照顺序执行：

表格

|阶段|配置项|执行时机|作用对象|核心作用|
|---|---|---|---|---|
|目标重标|`relabel_configs`|抓取目标被发现之后，发起指标抓取请求之前|服务发现生成的抓取目标的生命周期标签|过滤抓取目标、修改抓取地址、提取元数据为永久标签、调整抓取参数|
|指标重标|`metric_relabel_configs`|指标抓取完成之后，写入本地 TSDB 存储之前|抓取到的所有时序指标的标签|剔除无用指标、删除冗余标签、过滤高基数时序、降低存储压力|

> 补充：还有两个可选的 Relabeling 阶段：Alert Relabeling（告警发送前修改告警标签）、Remote Write Relabeling（远程写入前修改指标标签），将在后续对应章节讲解。

### 6.4.2 Relabeling 通用语法结构

无论是`relabel_configs`还是`metric_relabel_configs`，单条 Relabeling 规则的语法结构完全一致，为数组类型，多条规则按照从上到下的顺序依次执行，前一条规则的输出作为后一条规则的输入。

单条规则的标准结构：

```yaml
- source_labels: [ <label_name1>, <label_name2>, ... ] # 源标签列表，可选
  separator: ; # 源标签值的拼接分隔符，默认;
  regex: <regular_expression> # 正则表达式，匹配源标签拼接后的值，默认(.*)
  target_label: <label_name> # 目标标签，replace/keep/drop/hashmod动作必填
  replacement: <string> # 替换值，支持正则分组引用，默认$1
  action: <action_name> # 执行动作，默认replace
```

核心执行逻辑（默认 replace 动作）：

1. 将`source_labels`中指定的所有标签的值，按照`separator`拼接为一个字符串；
2. 使用`regex`正则表达式匹配拼接后的字符串，正则为 RE2 语法，全字符串匹配；
3. 若匹配成功，将`replacement`中的分组引用（$1、$2 等）替换为正则匹配的分组内容；
4. 将替换后的结果写入`target_label`指定的标签；
5. 若匹配失败，不执行任何操作，直接进入下一条规则。

### 6.4.3 核心动作详解

Relabeling 支持 7 类核心动作，覆盖所有标签处理场景，每类动作的语义、语法、示例与适用场景如下：

#### 6.4.3.1 `replace`（默认动作）

- **核心语义**：正则匹配源标签的值，将匹配结果替换后写入目标标签，是最常用的标签修改、添加动作。
- **标准示例 1：修改抓取地址**

    ```yaml
    # 将__address__的端口从8080修改为9100
    - source_labels: [__address__]
      regex: ([^:]+)(?::\d+)?
      replacement: ${1}:9100
      target_label: __address__
      action: replace
    ```
    
- **标准示例 2：提取元数据为永久标签**

    ```yaml
    # 从__meta_kubernetes_namespace中提取命名空间，写入namespace标签
    - source_labels: [__meta_kubernetes_namespace]
      target_label: namespace
      action: replace
    ```
    
- **适用场景**：标签添加、标签值修改、抓取参数调整、元数据提取。

#### 6.4.3.2 `keep`

- **核心语义**：仅保留**源标签值匹配正则表达式**的时序 / 目标，过滤掉不匹配的内容，是核心的白名单过滤动作。
- **标准示例 1：仅保留 prod 环境的实例**

    ```yaml
    - source_labels: [env]
      regex: prod
      action: keep
    ```
    
- **标准示例 2：仅保留带有 scrape=true 注解的 Pod**

    ```yaml
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      regex: "true"
      action: keep
    ```
    
- **适用场景**：抓取目标白名单过滤、无用指标剔除、高基数时序过滤，是控制时序基数的核心手段。

#### 6.4.3.3 `drop`

- **核心语义**：与 keep 相反，丢弃**源标签值匹配正则表达式**的时序 / 目标，保留不匹配的内容，是核心的黑名单过滤动作。
- **标准示例 1：丢弃 test 和 dev 环境的实例**

    ```yaml
    - source_labels: [env]
      regex: test|dev
      action: drop
    ```
    
- **标准示例 2：丢弃健康状态异常的 Consul 实例**

    ```yaml
    - source_labels: [__meta_consul_health]
      regex: critical|warning
      action: drop
    ```
    
- **适用场景**：抓取目标黑名单过滤、无用指标剔除、异常实例过滤。

#### 6.4.3.4 `hashmod`

- **核心语义**：对源标签拼接后的值进行哈希计算，取模后写入目标标签，用于抓取任务的分片、负载均衡，是大规模集群下 Prometheus 水平扩展的核心动作。
- **标准示例：按 instance 标签哈希取模，实现 4 实例分片采集**

    ```yaml
    # 对instance值哈希取模，结果写入临时标签
    - source_labels: [instance]
      modulus: 4
      target_label: __tmp_hash
      action: hashmod
    # 仅保留模等于0的实例，每个Prometheus实例负责1/4的目标
    - source_labels: [__tmp_hash]
      regex: "0"
      action: keep
    ```
    
- **适用场景**：大规模集群下 Prometheus 采集任务的水平分片、负载均衡，避免单实例采集压力过大。

#### 6.4.3.5 `labelmap`

- **核心语义**：正则匹配**标签名**（而非标签值），将匹配结果替换后作为新的标签名，原标签值保留，用于批量提取元数据标签、批量重命名标签。
- **标准示例 1：批量提取 Pod 的所有标签**

    ```yaml
    # 将__meta_kubernetes_pod_label_xxx开头的标签，重命名为k8s_label_xxx
    - regex: __meta_kubernetes_pod_label_(.+)
      replacement: k8s_label_${1}
      action: labelmap
    ```
    
- **标准示例 2：批量提取 Consul 的自定义标签**

    ```yaml
    - regex: __meta_consul_tag_(.+)
      replacement: consul_tag_${1}
      action: labelmap
    ```
    
- **适用场景**：批量提取服务发现的元数据标签、批量重命名标签、标签名的批量转换。

#### 6.4.3.6 `labeldrop`

- **核心语义**：正则匹配**标签名**，删除匹配到的标签，用于剔除冗余标签、降低时序基数。
- **标准示例：删除所有以 tmp_开头的临时标签**

    ```yaml
    - regex: tmp_.+
      action: labeldrop
    ```
    
- **适用场景**：删除无用标签、临时标签、冗余元数据标签，优化时序存储。

#### 6.4.3.7 `labelkeep`

- **核心语义**：与 labeldrop 相反，正则匹配**标签名**，仅保留匹配到的标签，删除所有不匹配的标签，是严格控制标签维度的核心动作。
- **标准示例：仅保留核心标签，删除其他所有标签**

    ```yaml
    - regex: job|instance|env|status|method|path
      action: labelkeep
    ```
    
- **适用场景**：严格控制指标的标签维度，避免高基数标签，大幅降低时序基数，是生产环境高基数治理的核心手段。

### 6.4.4 `relabel_configs` 深度应用

`relabel_configs`作用于抓取前的目标，核心应用场景分为三类，均有标准的生产级实现：

#### 1. 抓取目标过滤

通过 keep/drop 动作，实现抓取目标的白名单 / 黑名单控制，仅采集需要监控的目标，避免无效采集。

示例：Kubernetes SD 中，仅采集 monitoring 命名空间、带有 scrape=true 注解的 Pod：

```yaml
relabel_configs:
  - source_labels: [__meta_kubernetes_namespace]
    regex: monitoring
    action: keep
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    regex: "true"
    action: keep
```

#### 2. 抓取参数修改

通过修改生命周期标签，调整抓取的地址、路径、协议、参数，实现动态的抓取配置。

示例：从 Pod 注解中提取抓取参数，动态调整采集行为：

```yaml
relabel_configs:
  # 修改抓取路径
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
    regex: (.+)
    target_label: __metrics_path__
    action: replace
  # 修改抓取端口
  - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
    regex: ([^:]+)(?::\d+)?;(\d+)
    replacement: ${1}:${2}
    target_label: __address__
    action: replace
  # 修改抓取协议
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scheme]
    regex: (https?)
    target_label: __scheme__
    action: replace
```

#### 3. 元数据提取与标签标准化

通过 replace/labelmap 动作，将服务发现的元数据标签转换为永久标签，实现标签的标准化管理。

示例：从 Consul SD 中提取服务元数据，生成标准化标签：

```yaml
relabel_configs:
  - source_labels: [__meta_consul_service]
    target_label: job
    action: replace
  - source_labels: [__meta_consul_datacenter]
    target_label: datacenter
    action: replace
  - source_labels: [__meta_consul_node]
    target_label: node
    action: replace
  # 批量提取服务自定义标签
  - regex: __meta_consul_tag_(.+)
    action: labelmap
```

### 6.4.5 `metric_relabel_configs` 深度应用

`metric_relabel_configs`作用于抓取后的指标，存储前执行，是生产环境高基数治理、存储优化的核心手段，核心应用场景分为三类：

#### 1. 无用指标剔除

通过 keep/drop 动作，仅保留需要的指标，剔除低价值指标，大幅降低时序数量。

示例：Node Exporter 中仅保留核心资源指标，剔除其他无用指标：

```yaml
metric_relabel_configs:
  - source_labels: [__name__]
    regex: node_cpu_.+|node_memory_.+|node_filesystem_.+|node_network_.+
    action: keep
```

示例：丢弃 go_、process_开头的内置指标，仅保留业务指标：

```yaml
metric_relabel_configs:
  - source_labels: [__name__]
    regex: go_.+|process_.+
    action: drop
```

#### 2. 高基数时序过滤

通过 drop 动作，丢弃高基数标签的时序，从源头解决高基数问题。

示例：丢弃带有 user_id、order_id 等高基数标签的时序：

```yaml
metric_relabel_configs:
  - source_labels: [user_id, order_id]
    regex: .+
    action: drop
```

示例：丢弃 path 标签中包含动态参数的时序，避免基数爆炸：

```yaml
metric_relabel_configs:
  - source_labels: [path]
    regex: .*/api/user/\d+.*
    action: drop
```

#### 3. 标签优化与裁剪

通过 labeldrop/labelkeep 动作，删除冗余标签，严格控制标签维度，降低时序基数。

示例：删除所有临时标签：

```yaml
metric_relabel_configs:
  - regex: __tmp_.+
    action: labeldrop
```

示例：严格保留核心标签，删除其他所有标签，控制基数：

```yaml
metric_relabel_configs:
  - regex: job|instance|env|status|method|path
    action: labelkeep
```

### 6.4.6 核心最佳实践与常见坑

1. **规则执行顺序**：多条规则按照从上到下的顺序执行，需合理规划顺序：先过滤目标，再修改标签，最后提取元数据；
2. **正则匹配规范**：正则为 RE2 语法，全字符串匹配，而非子串匹配，需注意 ^ 和 $ 的隐含逻辑；
3. **临时标签规范**：中间计算结果使用`__tmp_`前缀的标签，Relabeling 执行完成后会被自动丢弃，不会污染永久标签；
4. **高基数治理前置**：优先使用`relabel_configs`过滤抓取目标，再使用`metric_relabel_configs`过滤指标，从源头减少无效采集与存储；
5. **labelkeep 慎用**：labelkeep 会删除所有不匹配的标签，包括 job、instance 等核心标签，需确保正则包含所有必要标签；
6. **避免过度 Relabeling**：过多的规则会增加 Prometheus 的 CPU 开销，需保持规则简洁高效，避免冗余逻辑。

---
