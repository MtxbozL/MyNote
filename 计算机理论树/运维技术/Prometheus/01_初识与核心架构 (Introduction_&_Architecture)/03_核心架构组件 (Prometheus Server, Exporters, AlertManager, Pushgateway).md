
Prometheus 采用模块化的分层架构设计，各组件职责单一、协同工作，构成完整的监控、存储、告警、可视化全链路体系。核心架构组件包括 Prometheus Server、Exporters、AlertManager、Pushgateway 四大核心模块，各组件的核心职责与协同逻辑如下。

### 1.3.1 Prometheus Server

Prometheus Server 是整个监控体系的核心中枢，负责指标的抓取、存储、规则计算与查询响应，其内部可分为四个核心子模块：

1. **Retrieval（抓取模块）**：负责按照配置的抓取规则，通过 HTTP 协议从监控目标（Exporters、Pushgateway、业务埋点接口等）拉取时序指标数据；
2. **TSDB（时序数据库模块）**：负责时序数据的持久化存储，采用本地磁盘存储的时序数据库引擎，支持高吞吐的时序数据写入与高效的多维查询；
3. **Rule Manager（规则管理模块）**：负责定期执行配置的记录规则（Recording Rule）与告警规则（Alerting Rule），完成预聚合计算与告警阈值的评估；
4. **HTTP Server 模块**：提供标准化的 HTTP 查询接口，对外暴露 PromQL 查询能力，支撑 Grafana 可视化、告警推送与第三方系统集成。

### 1.3.2 Exporters

Exporters 是 Prometheus 生态的指标暴露组件，负责将被监控对象的运行状态数据转换为 Prometheus 兼容的时序指标格式，并通过 HTTP 接口暴露，供 Prometheus Server 拉取。

Exporters 覆盖了全场景的监控对象，核心分类包括：

- 基础设施类 Exporters：如 Node Exporter（主机 / 服务器指标监控）、cAdvisor（容器资源指标监控）；
- 黑盒探测类 Exporters：如 Blackbox Exporter（网络、端口、HTTP 接口的黑盒探测）；
- 中间件与数据库类 Exporters：如 MySQL Exporter、Redis Exporter、Kafka Exporter 等；
- 自定义业务埋点：通过 Prometheus 提供的客户端库（Golang、Java、Python 等），在业务代码中实现自定义指标的埋点与暴露。

### 1.3.3 AlertManager

AlertManager 是 Prometheus 生态的告警管理核心组件，负责接收 Prometheus Server 推送的告警事件，并完成告警的分组、聚合、路由、抑制、静默与最终的通知分发。

其核心定位是解决告警风暴、实现精细化的告警管理，支持对接 Email、企业微信、钉钉、Slack 等多种通知渠道，同时支持自定义 Webhook 对接自研告警管理系统。

### 1.3.4 Pushgateway

Pushgateway 是 Prometheus 生态的推送网关组件，作为 Pull 模型的补充，用于解决**短生命周期、临时批处理任务**的指标采集问题。

此类任务的运行周期短于 Prometheus 的抓取间隔，无法通过 Pull 模型完成指标采集，需通过 Pushgateway 将指标临时推送并缓存，再由 Prometheus Server 从 Pushgateway 统一拉取。

---
