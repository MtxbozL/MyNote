
Prometheus 内置了数十种主流平台的服务发现适配器，覆盖服务注册中心、容器编排平台、公有云厂商三大类场景。本节重点讲解生产环境最核心的 Consul SD 与 Kubernetes SD，简要介绍公有云 EC2 SD。

### 6.3.1 Consul SD

Consul 是 HashiCorp 开源的分布式服务注册与发现中心，是传统微服务架构中最主流的服务注册方案。Prometheus 原生支持 Consul SD，可自动从 Consul Catalog 中拉取服务实例列表与健康状态，动态更新抓取目标。

#### 6.3.1.1 核心工作原理

1. 服务提供者启动时，将自身的地址、端口、标签、健康检查规则注册到 Consul Server；
2. Consul Server 持续维护服务实例的健康状态，自动剔除不健康的实例；
3. Prometheus 通过 Consul SD 配置，定期调用 Consul Catalog API，拉取指定服务的健康实例列表；
4. Prometheus 将 Consul 中的服务元数据转换为`__meta_consul_`前缀的生命周期标签，供 Relabeling 处理；
5. 自动更新抓取目标，无需热加载，实现监控与服务生命周期的完全同步。

#### 6.3.1.2 标准配置示例

```yaml
scrape_configs:
  - job_name: "consul_sd_microservice"
    consul_sd_configs:
      - server: "consul-server.prod.svc:8500" # Consul Server地址
        datacenter: "dc1" # 数据中心，可选
        scheme: "http" # 协议，生产环境推荐https
        token: "your_consul_acl_token" # ACL Token，生产环境必填
        services: ["api-gateway", "user-service", "order-service"] # 指定要发现的服务，不填则发现所有服务
        tags: ["prod", "monitoring"] # 按标签过滤实例，可选
    relabel_configs:
      # 提取服务名称作为job标签
      - source_labels: [__meta_consul_service]
        target_label: job
      # 仅保留健康状态为passing的实例
      - source_labels: [__meta_consul_health]
        regex: passing
        action: keep
      # 提取实例ID作为instance标签
      - source_labels: [__meta_consul_service_id]
        target_label: instance
      # 提取环境标签
      - source_labels: [__meta_consul_tags]
        regex: ",env=([^,]+),"
        replacement: "${1}"
        target_label: env
```

#### 6.3.1.3 核心元数据标签

Consul SD 会为每个实例生成`__meta_consul_`前缀的生命周期标签，核心包括：

|标签名|核心含义|
|---|---|
|`__meta_consul_service`|服务名称|
|`__meta_consul_service_id`|服务实例唯一 ID|
|`__meta_consul_address`|服务实例 IP 地址|
|`__meta_consul_port`|服务实例端口|
|`__meta_consul_tags`|服务实例的标签列表，逗号分隔|
|`__meta_consul_health`|服务实例的健康状态（passing/warning/critical）|
|`__meta_consul_datacenter`|Consul 数据中心名称|
|`__meta_consul_node`|服务所在的 Consul 节点名称|

### 6.3.2 Kubernetes SD

Kubernetes SD 是 Prometheus 为云原生环境设计的核心服务发现机制，原生适配 Kubernetes API，可自动发现集群内的 node、pod、service、endpoints、ingress 等资源，完美解决了 Pod IP 动态漂移、服务实例频繁销毁重建的核心痛点，是 Kubernetes 环境下监控的必备能力。

#### 6.3.2.1 核心工作原理

1. Prometheus 通过 Kubernetes 的 ServiceAccount 权限，调用 Kubernetes API Server，监听指定资源的变更事件；
2. 当 Kubernetes 资源发生创建、销毁、更新时，Prometheus 实时获取最新的资源列表，自动更新抓取目标；
3. Prometheus 将 Kubernetes 资源的元数据转换为`__meta_kubernetes_`前缀的生命周期标签，供 Relabeling 处理；
4. 完全适配 Kubernetes 的声明式 API，无需手动维护任何抓取目标，实现监控与服务生命周期的完全同步。

#### 6.3.2.2 前置权限要求

Prometheus 在 Kubernetes 集群内运行时，必须绑定对应的 ClusterRole，授予核心资源的 get、list、watch 权限，标准 RBAC 配置如下：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-k8s-sd
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/metrics
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics", "/metrics/cadvisor"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-k8s-sd
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus-k8s-sd
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
```

#### 6.3.2.3 核心 Role 类型与配置示例

Kubernetes SD 支持 5 种 Role，分别对应不同的 Kubernetes 资源，覆盖全场景监控需求：

##### 1. Role: node

用于发现 Kubernetes 集群的 Node 节点，适配 Node Exporter、kubelet、cAdvisor 的监控，自动发现所有 Node 节点的地址。

```yaml
scrape_configs:
  - job_name: "k8s-sd-nodes"
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - source_labels: [__meta_kubernetes_node_name]
        target_label: node_name
      - source_labels: [__meta_kubernetes_node_label_kubernetes_io_hostname]
        target_label: hostname
```

##### 2. Role: pod

用于发现 Kubernetes 集群内的 Pod 实例，适配业务微服务、Sidecar 容器的监控，自动发现 Pod 的 IP、端口、元数据，是业务容器监控的核心 Role。

```yaml
scrape_configs:
  - job_name: "k8s-sd-pods"
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # 仅保留带有prometheus.io/scrape=true注解的Pod
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        regex: "true"
        action: keep
      # 从注解中提取metrics_path，默认/metrics
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        regex: "(.+)"
        target_label: __metrics_path__
        replacement: "${1}"
      # 从注解中提取端口，拼接抓取地址
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        regex: "([^:]+)(?::\\d+)?;(\\d+)"
        replacement: "${1}:${2}"
        target_label: __address__
      # 提取核心元数据为永久标签
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod_name
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
```

##### 3. 其他 Role 说明

- **Role: service**：发现 Kubernetes 的 Service 资源，适配黑盒探测、Service 端口监控；
- **Role: endpoints**：发现 Kubernetes 的 Endpoints 资源，适配 Service 后端 Pod 实例的精细化监控；
- **Role: ingress**：发现 Kubernetes 的 Ingress 资源，适配 HTTP/HTTPS 接口的黑盒可用性探测。

#### 6.3.2.4 核心元数据标签

Kubernetes SD 为每个资源生成`__meta_kubernetes_`前缀的生命周期标签，核心包括：

- 通用标签：`__meta_kubernetes_namespace`（资源所在命名空间）、`__meta_kubernetes_resource_name`（资源名称）；
- Pod 专属：`__meta_kubernetes_pod_name`、`__meta_kubernetes_pod_ip`、`__meta_kubernetes_pod_label_xxx`（Pod 的标签）、`__meta_kubernetes_pod_annotation_xxx`（Pod 的注解）；
- Service 专属：`__meta_kubernetes_service_name`、`__meta_kubernetes_service_label_xxx`、`__meta_kubernetes_service_annotation_xxx`；
- Node 专属：`__meta_kubernetes_node_name`、`__meta_kubernetes_node_label_xxx`、`__meta_kubernetes_node_address_InternalIP`。

### 6.3.3 EC2 SD

AWS EC2 SD 是 Prometheus 内置的公有云服务发现机制，可自动发现 AWS EC2 实例，动态更新抓取目标，适配 AWS 云环境的监控需求。核心原理是调用 AWS EC2 API 拉取实例列表，转换为`__meta_ec2_`前缀的生命周期标签，通过 Relabeling 过滤目标。

标准配置示例：

```yaml
scrape_configs:
  - job_name: "aws-ec2-sd"
    ec2_sd_configs:
      - region: ap-southeast-1
        access_key: your_aws_access_key
        secret_key: your_aws_secret_key
        port: 9100
        filters:
          - name: tag:env
            values: ["prod"]
```

适用场景：AWS 云环境的 EC2 实例监控，自动同步 EC2 的扩缩容、启停状态，无需手动维护目标列表。阿里云、腾讯云等公有云厂商均提供了适配 Prometheus 的服务发现插件，用法与 EC2 SD 一致。

---