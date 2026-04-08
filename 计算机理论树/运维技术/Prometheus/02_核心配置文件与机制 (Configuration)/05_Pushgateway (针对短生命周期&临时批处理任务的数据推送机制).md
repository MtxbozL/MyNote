
Pushgateway 是 Prometheus 生态的核心补充组件，专门解决**短生命周期、临时批处理任务**的指标采集痛点，是 Pull 模型的辅助方案，不可作为核心采集架构使用。

### 2.6.1 核心工作机制

短生命周期任务（如定时 Cron 任务、一次性批处理任务、CI/CD 流水线任务）的运行周期通常短于 Prometheus 的抓取间隔，无法被 Pull 模型稳定采集。Pushgateway 作为中间缓存层，实现了推送 - 拉取的中转：

1. 临时任务运行时，通过 HTTP 请求将指标推送到 Pushgateway，Pushgateway 将指标持久化缓存；
2. Prometheus 通过固定的抓取 Job，从 Pushgateway 拉取所有缓存的指标数据；
3. 即使临时任务已终止，其推送的指标仍可被 Prometheus 采集到，避免指标丢失。

### 2.6.2 部署与基础配置

#### 1. Pushgateway 部署

提供与第一章一致的两种标准化部署方式：

- **二进制部署**：从 Prometheus 官方下载地址获取对应架构的二进制包，解压后直接启动，默认监听 9091 端口：

    ```bash
    tar -zxvf pushgateway-<version>.linux-<arch>.tar.gz
    cd pushgateway-<version>.linux-<arch>
    ./pushgateway
    ```
    
- **Docker 快速部署**：
    
    ```bash
    docker run -d -p 9091:9091 --name pushgateway prom/pushgateway:latest
    ```

#### 2. Prometheus 侧抓取配置

在`scrape_configs`中添加固定 Job，抓取 Pushgateway 的指标接口：

```yaml
scrape_configs:
  - job_name: "pushgateway"
    scrape_interval: 15s
    static_configs:
      - targets: ["<pushgateway-ip>:9091"]
        labels:
          module: metrics-gateway
```

### 2.6.3 指标推送规范

#### 1. 推送接口格式

Pushgateway 的标准推送接口格式如下，通过 URL 路径指定指标的分组标签，保证指标的唯一性：

```bash
http://<pushgateway-ip>:9091/metrics/job/<job_name>/instance/<instance_name>/<label1>/<value1>/<label2>/<value2>
```

- 必须指定`job`标签，作为指标的核心分组标识；
- 推荐指定`instance`标签，区分不同的任务实例，避免指标覆盖；
- 可通过 URL 路径追加任意自定义标签，实现多维度分组。

#### 2. 标准推送示例

- **curl 命令推送**：适用于 Shell 脚本、Cron 任务等轻量场景：
    
    ```bash
    cat <<EOF | curl --data-binary @- http://127.0.0.1:9091/metrics/job/batch_task/instance/cron_job_01
    # TYPE batch_task_execution_total counter
    batch_task_execution_total{status="success"} 1
    batch_task_execution_total{status="failed"} 0
    # TYPE batch_task_execution_duration_seconds gauge
    batch_task_execution_duration_seconds 2.35
    EOF
    ```
    
- **客户端库推送**：Prometheus 提供 Go、Java、Python 等多语言客户端库，可在业务代码中实现标准化的指标推送，适配复杂批处理任务场景。

### 2.6.4 适用边界与使用禁忌

#### 严格适用场景

仅适用于短生命周期、不可持续暴露 /metrics 接口的任务：

- 定时 Cron 任务、一次性批处理任务、离线计算任务；
- CI/CD 流水线任务、数据同步任务、备份任务；
- 无固定网络地址、仅支持出站访问的边缘任务。

#### 绝对禁止场景

1. 禁止作为长生命周期服务、基础设施、中间件的核心采集方案，替代原生 Pull 模型；
2. 禁止用于多实例部署的服务指标采集，会导致指标覆盖、无法区分实例的问题；
3. 禁止用于高基数指标的推送，会导致 Pushgateway 内存溢出、Prometheus 抓取压力激增。

#### 核心注意事项

1. Pushgateway 默认**不会自动删除**推送的指标，即使任务已终止，指标会永久缓存，需通过`DELETE`请求手动删除，或启动时添加`--persistence.interval`参数设置指标过期时间；
2. Pushgateway 仅做指标透传，不会校验指标的时序唯一性，重复推送相同标签的指标会导致数据覆盖；
3. Pushgateway 为有状态服务，存在单点故障风险，生产环境需做好高可用部署与数据持久化。

---

## 本章小结

本章完成了 Prometheus 核心配置与运行机制的系统性学习，核心内容包括：

1. 掌握了 Prometheus 核心配置文件的 YAML 语法规范与顶层结构，建立了配置与架构模块的对应关系；
2. 深入理解了全局配置核心参数的含义、生效范围与生产环境选型策略，可完成标准化的全局配置编写；
3. 熟练掌握了抓取配置的核心逻辑，可通过静态配置完成固定目标的采集任务，理解了 Job 与采集目标的分组逻辑；
4. 明确了规则文件的加载机制、路径配置规范与合法性校验方法，掌握了生产环境配置变更前的校验流程；
5. 掌握了 Prometheus 热加载的两种标准实现方式、生效前提与限制，可完成生产环境无中断的配置更新；
6. 深入理解了 Pushgateway 的工作机制、部署配置、推送规范，严格界定了其适用边界与使用禁忌，避免了架构误用。

本章为后续深入学习时序数据模型、PromQL 查询语言、动态服务发现等核心内容建立了配置基础与运行环境支撑。