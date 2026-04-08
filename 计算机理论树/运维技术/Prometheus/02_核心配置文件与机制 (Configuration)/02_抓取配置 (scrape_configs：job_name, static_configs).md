
`scrape_configs`是 Prometheus 指标采集的核心配置模块，为数组类型，每个元素对应一个**抓取 Job**。Job 是 Prometheus 对一组采集目标的逻辑分组，同一 Job 内的所有采集目标共享一套抓取配置，每个 Job 必须通过`job_name`指定全局唯一的名称，该名称会作为`job`标签自动附加到该 Job 采集的所有时序数据中。

### 2.3.1 Job 核心基础参数

Job 级参数可覆盖全局配置的对应默认值，实现不同采集任务的精细化配置，核心参数如下：

|参数名|核心作用|生效规则|
|---|---|---|
|`job_name`|Job 的唯一标识，作为时序数据的`job`标签值|必填，全局唯一，不可重复|
|`scrape_interval`|该 Job 专属的抓取间隔|覆盖全局`scrape_interval`配置|
|`scrape_timeout`|该 Job 专属的抓取超时时间|覆盖全局`scrape_timeout`配置|
|`metrics_path`|采集目标的指标暴露路径|默认为`/metrics`，可自定义为业务适配的路径（如`/actuator/prometheus`）|
|`scheme`|抓取请求的协议类型|默认为`http`，可选`https`用于加密采集场景|
|`basic_auth`/`bearer_token`|抓取请求的身份认证配置|用于需鉴权的指标接口，如 Kubernetes 组件、加密业务接口|

### 2.3.2 静态目标配置（static_configs）

静态配置是 Prometheus 最基础的目标发现方式，通过手动指定采集目标的网络地址列表，实现固定采集目标的配置，适用于地址固定的基础设施、中间件、单机服务等场景。

#### 核心配置结构

- `targets`：数组类型，元素为`IP:Port`格式的采集目标地址，必填；
- `labels`：键值对类型，为该组目标采集的所有时序数据附加自定义标签，可选。

#### 标准配置示例

```yaml
scrape_configs:
  # Job1：Prometheus自身指标采集
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
        labels:
          module: monitoring
          role: prometheus-server

  # Job2：主机节点指标采集（Node Exporter）
  - job_name: "node_exporter"
    scrape_interval: 10s
    scrape_timeout: 5s
    static_configs:
      - targets: ["192.168.1.10:9100", "192.168.1.11:9100", "192.168.1.12:9100"]
        labels:
          idc: guangzhou-az1
          business: infrastructure
```

> 注：每个采集目标会自动生成`instance`标签，值为`targets`中配置的`IP:Port`地址，与`job`标签共同构成时序数据的基础维度标识。

---
