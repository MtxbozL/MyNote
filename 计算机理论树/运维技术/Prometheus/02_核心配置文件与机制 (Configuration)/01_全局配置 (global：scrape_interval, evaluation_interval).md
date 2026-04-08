## 本章学习目标

1. 掌握 Prometheus 核心配置文件`prometheus.yml`的整体结构与语法规范
2. 理解全局配置的核心参数含义、生效范围与生产环境选型策略
3. 熟练掌握抓取配置的核心逻辑，完成静态采集任务的标准化配置
4. 明确规则文件的加载机制与合法性校验方法
5. 掌握 Prometheus 热加载的实现方式、生效前提与生产级操作规范
6. 深入理解 Pushgateway 的工作机制、配置方法、适用边界与使用禁忌

---

## 2.1 配置文件整体规范

Prometheus 的核心行为完全由 YAML 格式的配置文件定义，默认配置文件名为`prometheus.yml`。配置文件采用严格的 YAML 语法规范，大小写敏感，通过缩进实现层级结构，禁止使用 Tab 键缩进。

核心配置文件的顶层结构分为五大核心模块，与本章内容一一对应：

```yaml
# 全局配置：全实例生效的默认参数
global:
  [配置项]

# 规则文件配置：指定记录规则与告警规则的加载路径
rule_files:
  [ - 文件路径 ]

# 抓取配置：定义指标采集任务与目标
scrape_configs:
  [ - 抓取Job配置 ]

# Alertmanager配置：定义告警接收端的对接参数
alerting:
  [ alertmanager配置 ]

# 远程读写配置：定义时序数据的远程存储对接参数
remote_write:
  [ - 远程写配置 ]
remote_read:
  [ - 远程读配置 ]
```

> 注：`alerting`与远程读写配置将在后续对应章节详细讲解，本章聚焦前四大核心模块与热加载、Pushgateway 机制。

---

## 2.2 全局配置（global）

全局配置是 Prometheus 实例的顶层默认参数，对所有抓取任务、规则评估任务生效；若子模块（如单个抓取 Job）未单独配置对应参数，将直接继承全局配置的默认值。

### 2.2.1 核心参数详解

|参数名|类型|默认值|核心作用与说明|
|---|---|---|---|
|`scrape_interval`|时间 duration|15s|全局默认的指标抓取间隔，即 Prometheus Server 向采集目标发起拉取请求的时间周期。该参数直接平衡指标精度与存储、计算压力：基础设施监控推荐 10s-15s，业务指标监控推荐 30s-1m，低优先级离线任务推荐 1m-5m。|
|`evaluation_interval`|时间 duration|1m|全局规则评估间隔，即 Prometheus Server 定期执行记录规则（Recording Rule）与告警规则（Alerting Rule）的时间周期。该参数不可被子模块覆盖，对所有规则文件全局生效。|
|`scrape_timeout`|时间 duration|10s|全局默认的抓取超时时间，即单次拉取指标请求的最大等待时长，超过该时长将判定为抓取失败。该值必须小于对应 Job 的`scrape_interval`，否则会出现请求重叠。|
|`external_labels`|键值对 map|无|全局外部标签，会自动附加到该实例所有本地存储的时序数据、对外发送的告警事件、联邦集群同步的数据、远程读写的时序数据中。核心作用是唯一标识 Prometheus 实例，区分多集群、多环境、多可用区，生产环境必填，示例：`cluster: prod-az1`、`env: production`。|

### 2.2.2 标准配置示例

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 30s
  scrape_timeout: 10s
  external_labels:
    cluster: production-az1
    env: prod
    prometheus_instance: prometheus-01
```

---
