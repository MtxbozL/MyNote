
## 10.3 与 Prometheus 生态整合

### 10.3.1 整合定位与核心价值

Zabbix 与 Prometheus 并非替代关系，而是互补的监控体系，二者整合的核心价值在于：**复用企业已有的 Prometheus Exporter 生态，无需重复开发采集脚本，实现 Zabbix 大一统监控平台对容器、云原生、中间件场景的全覆盖**，解决监控体系碎片化、多平台维护成本高的痛点。

Zabbix 6.0+ LTS 版本原生支持 Prometheus 指标格式的解析与采集，无需额外插件或组件，开箱即用。

### 10.3.2 核心采集原理

Prometheus Exporter 通过`/metrics`接口暴露 Prometheus 文本格式的时序指标，Zabbix 通过以下两种原生方式实现指标采集与解析：

1. **HTTP Agent 采集模式**：通过 HTTP Agent 监控项，直接抓取 Exporter 的`/metrics`接口，原生解析 Prometheus 文本格式，通过正则表达式、LLD 规则提取目标指标，是最常用的整合方式；
2. **Zabbix Agent 2 Prometheus 插件模式**：通过 Agent 2 内置的 Prometheus 采集插件，本地或远程抓取 Exporter 指标，支持预聚合、标签过滤、自动发现，性能更强，适用于高频采集场景。

核心适配能力：

- 原生支持 Prometheus 文本格式解析，兼容 Gauge、Counter、Histogram、Summary 四大核心指标类型；
- 支持 Prometheus 标签过滤与匹配，可基于标签提取特定维度的指标；
- 支持通过 LLD 低级别发现，自动发现 Prometheus 指标的标签维度，批量生成监控项与触发器；
- 支持 Counter 类型指标的增量计算、速率计算，适配 Prometheus 累计型指标的监控需求。

### 10.3.3 标准化整合实战：抓取 Node Exporter 指标

Node Exporter 是 Prometheus 生态中最常用的主机监控 Exporter，以下为 Zabbix 通过 HTTP Agent 抓取 Node Exporter 指标的标准化配置流程。

#### 10.3.3.1 前置准备

1. 目标主机部署 Node Exporter，默认端口 9100，确保`http://<目标主机IP>:9100/metrics`可正常访问，返回 Prometheus 格式指标；
2. Zabbix Server 与目标主机 9100 端口网络可达，或通过 Agent 2 代理访问。

#### 10.3.3.2 单指标采集配置

以采集主机 CPU 空闲率指标`node_cpu_seconds_total{mode="idle"}`为例，配置 HTTP Agent 监控项：

1. 进入主机监控项配置页面，创建新监控项；
2. 类型选择「HTTP 代理」，名称填写「CPU 空闲时间总量」；
3. URL 填写`http://<目标主机IP>:9100/metrics`，请求方法选择`GET`；
4. 「已接收数据的预处理」添加预处理规则：
    
    - 规则类型：「Prometheus 模式匹配」
    - 参数 1：指标名称`node_cpu_seconds_total`
    - 参数 2：标签过滤`mode="idle"`
    - 参数 3：值输出，默认`value`
    
5. 数据类型选择「数字（浮点数）」，更新周期配置 60s，单位`s`；
6. 测试验证，确认可正常获取指标值。

针对 Counter 类型的累计指标，可添加「简单更改」预处理规则，选择「每秒变化率」，计算指标的增长速率，适配 Prometheus Counter 类型的监控需求。

#### 10.3.3.3 LLD 自动发现批量监控

通过 LLD 低级别发现规则，自动发现 Node Exporter 的磁盘分区、网卡等维度，批量生成监控项：

1. 进入主机 LLD 规则配置页面，创建新发现规则；
2. 类型选择「HTTP 代理」，名称填写「Node Exporter 磁盘分区自动发现」；
3. URL 填写`http://<目标主机IP>:9100/metrics`，请求方法`GET`；
4. 「已接收数据的预处理」添加规则：
    
    - 规则类型：「Prometheus 模式匹配到 JSON」
    - 参数 1：指标名称`node_filesystem_size_bytes`
    - 参数 2：标签过滤，留空获取所有分区
    
5. 过滤器配置：过滤掉虚拟文件系统，仅保留真实磁盘分区；
6. 创建监控项原型、触发器原型，使用 LLD 宏`{#device}`、`{#fstype}`、`{#mountpoint}`动态填充参数，实现每个分区的自动监控。

### 10.3.4 生产环境整合最佳实践

1. **采集模式选型**：单主机少量指标采集使用 HTTP Agent 模式；大规模、高频指标采集使用 Agent 2 Prometheus 插件模式，性能更优；
2. **指标过滤原则**：仅采集需要的核心指标，禁止全量抓取 Exporter 的所有指标，避免给 Zabbix Server 带来过大的 NVPS 压力；
3. **LLD 规则优化**：严格配置过滤器，仅保留需要监控的维度，避免生成大量无用监控项；拉长 LLD 执行周期，最低 30 分钟，非动态资源设置为 24 小时；
4. **指标类型适配**：Counter 类型指标必须使用速率计算，Gauge 类型指标直接使用原始值，Histogram/Summary 类型仅采集核心分位数指标；
5. **模板化复用**：将 Prometheus 采集规则、LLD 规则、监控项封装为标准化模板，实现批量部署，无需逐台配置；
6. **性能约束**：单台主机通过 Prometheus Exporter 采集的监控项数量不超过 500 个，避免 NVPS 过载。
