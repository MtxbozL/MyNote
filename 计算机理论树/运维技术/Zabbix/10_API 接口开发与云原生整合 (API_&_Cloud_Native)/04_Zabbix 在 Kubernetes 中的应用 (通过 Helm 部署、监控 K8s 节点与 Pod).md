
## 10.5 Zabbix 在 Kubernetes 中的应用

### 10.5.1 场景定位与核心痛点

Kubernetes（简称 K8s）是云原生应用的核心编排平台，其动态扩缩容、容器生命周期短、IP 动态变化、微服务架构的特性，给传统监控带来了核心痛点：

1. 容器 / Pod 动态扩缩容，IP 频繁变化，传统静态主机监控模式无法适配；
2. 监控对象粒度从主机下沉到 Pod、容器、容器内应用、K8s 集群资源，维度大幅增加；
3. 微服务架构下，实例数量动态变化，手动配置监控完全无法落地；
4. K8s 集群本身的组件、节点、命名空间、Deployment 等资源需要全维度监控。

Zabbix 6.0+ LTS 版本通过 Agent 2 原生 K8s 插件、Helm Chart 标准化部署、LLD 自动发现、主动注册机制，完美适配 K8s 云原生场景，实现 K8s 集群全栈监控。

### 10.5.2 标准化部署：基于 Helm Chart

Helm 是 K8s 的包管理工具，Zabbix 官方提供了标准化的 Helm Chart，是 K8s 环境中部署 Zabbix 的首选方式，支持一键部署 Zabbix 全栈组件，兼容高可用、持久化、分布式架构。

#### 10.5.2.1 核心部署架构

Helm Chart 部署的标准架构包含以下核心组件，可按需开启 / 关闭：

1. **Zabbix Server**：核心服务端，支持多实例原生 HA 集群，通过 StatefulSet 部署；
2. **Zabbix Web Frontend**：Web 管理界面，通过 Deployment 部署，支持多实例横向扩展，搭配 Ingress 对外提供访问；
3. **数据库**：支持 PostgreSQL/MySQL，通过 StatefulSet 部署，搭配持久化卷存储数据，支持高可用集群；
4. **Zabbix Agent 2**：通过 DaemonSet 部署在每个 K8s 节点上，采集节点与节点上的容器指标；
5. **Zabbix Proxy**：可选组件，通过 Deployment/DaemonSet 部署，用于大规模 K8s 集群的分布式采集，减轻中心 Server 压力。

#### 10.5.2.2 标准化部署流程

1. **环境准备**：K8s 集群版本 1.24+，Helm 3.0+，集群配置了默认 StorageClass，用于持久化存储；
2. **添加 Zabbix 官方 Helm 仓库**：
    
    bash
    
    运行
    
    ```
    helm repo add zabbix https://zabbix-community.github.io/helm-charts
    helm repo update
    ```
    
3. **自定义配置文件**：创建`values.yaml`文件，配置组件开启 / 关闭、持久化、资源限制、高可用、Ingress 等核心参数，核心配置示例：
    
    yaml
    
    ```
    # Zabbix Server配置
    zabbixServer:
      enabled: true
      replicaCount: 2
      image:
        repository: zabbix/zabbix-server-pgsql
        tag: 7.0-latest
      resources:
        limits:
          cpu: 4
          memory: 8Gi
        requests:
          cpu: 2
          memory: 4Gi
    
    # 数据库配置（PostgreSQL）
    postgresql:
      enabled: true
      auth:
        postgresPassword: "ZabbixDBPass123"
        username: "zabbix"
        password: "ZabbixUserPass123"
        database: "zabbix"
      primary:
        persistence:
          enabled: true
          size: 100Gi
          storageClass: "nfs-client"
    
    # Zabbix Web配置
    zabbixWeb:
      enabled: true
      replicaCount: 2
      image:
        repository: zabbix/zabbix-web-nginx-pgsql
        tag: 7.0-latest
      ingress:
        enabled: true
        hosts:
          - host: zabbix-k8s.example.com
            paths:
              - path: /
                pathType: Prefix
        tls:
          - hosts:
              - zabbix-k8s.example.com
            secretName: zabbix-tls
    
    # Zabbix Agent 2 DaemonSet
    zabbixAgent2:
      enabled: true
      image:
        repository: zabbix/zabbix-agent2
        tag: 7.0-latest
    ```
    
4. **执行部署**：通过 Helm 安装 Zabbix 集群
    
    bash
    
    运行
    
    ```
    helm install zabbix zabbix/zabbix -f values.yaml -n zabbix --create-namespace
    ```
    
5. **部署验证**：查看 Pod 运行状态，通过 Ingress 配置的域名访问 Zabbix Web 前端，确认部署成功。

### 10.5.3 K8s 全栈监控实现

Zabbix 通过 Agent 2 原生 Kubernetes 插件，实现 K8s 集群全维度监控，无需额外组件，开箱即用。

#### 10.5.3.1 前置权限配置

Agent 2 访问 K8s API 需要对应的 RBAC 权限，需创建 ServiceAccount、ClusterRole、ClusterRoleBinding，配置示例：

yaml

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: zabbix-agent-k8s-monitor
rules:
  - apiGroups: [""]
    resources: ["nodes", "pods", "services", "endpoints", "namespaces", "persistentvolumeclaims", "events"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "daemonsets", "statefulsets", "replicasets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["batch"]
    resources: ["jobs", "cronjobs"]
    verbs: ["get", "list", "watch"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: zabbix-agent-k8s-monitor
  namespace: zabbix
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: zabbix-agent-k8s-monitor
subjects:
  - kind: ServiceAccount
    name: zabbix-agent-k8s-monitor
    namespace: zabbix
roleRef:
  kind: ClusterRole
  name: zabbix-agent-k8s-monitor
  apiGroup: rbac.authorization.k8s.io
```

在 Agent 2 的 DaemonSet 配置中，关联该 ServiceAccount，即可获得 K8s API 的访问权限。

#### 10.5.3.2 核心监控维度

Agent 2 K8s 插件原生支持以下核心监控维度，通过 LLD 自动发现批量生成监控项：

1. **K8s 集群层面**：集群节点总数、就绪节点数、不可调度节点数、命名空间数量、Pod 总数、运行 / 异常 Pod 数、API Server 状态、调度器状态、etcd 状态；
2. **节点层面**：节点 CPU / 内存 / 磁盘使用率、节点状态、节点调度状态、节点运行 Pod 数、容器运行时状态、Kubelet 状态；
3. **工作负载层面**：Deployment/StatefulSet/DaemonSet 的副本数、就绪副本数、更新进度、可用率，通过 LLD 自动发现所有工作负载，批量生成监控项与触发器；
4. **Pod / 容器层面**：Pod 运行状态、重启次数、CPU / 内存使用率、网络流量、容器健康状态、重启告警、OOM 告警，通过 LLD 自动发现所有 Pod，适配动态扩缩容；
5. **网络与存储层面**：Service 状态、Endpoint 可用性、PVC 使用率、PV 状态、StorageClass 可用性；
6. **K8s 事件监控**：采集 K8s 集群的 Warning 级别事件，如 Pod OOM、镜像拉取失败、调度失败，自动触发告警。

#### 10.5.3.3 动态适配能力

针对 K8s 动态扩缩容的特性，Zabbix 通过以下机制实现完美适配：

1. **LLD 低级别发现**：定期自动发现新增的 Pod、Deployment、Service，自动生成对应的监控项与触发器，无需人工干预；
2. **主动注册机制**：Pod 启动后，内置的 Agent 2 主动向 Zabbix Server 注册，自动完成监控配置上线；
3. **资源丢失自动处理**：Pod 销毁后，对应的监控项自动禁用 / 删除，避免无效监控配置；
4. **标签过滤**：支持基于 K8s 标签过滤监控对象，仅监控指定命名空间、指定业务标签的工作负载，减少无效监控。

### 10.5.4 生产环境最佳实践

1. **分布式采集架构**：超过 50 个节点的 K8s 集群，优先部署 Zabbix Proxy，将采集压力下沉到 Proxy，减轻中心 Server 压力；
2. **主动模式优先**：所有 Agent 2、Proxy 均采用主动模式，完美适配 Pod IP 动态变化的场景，无需静态配置 IP；
3. **监控分级原则**：核心业务 Pod、K8s 核心组件采用高频采集（30s-1min），非核心资源采用低频采集（5-10min），控制 NVPS 总量；
4. **持久化规范**：数据库必须使用高性能持久化卷，禁止使用 emptyDir 等临时存储，避免数据丢失；Zabbix Server、Proxy 的配置文件、日志需配置持久化；
5. **高可用配置**：生产环境必须部署 2 + 节点的 Zabbix Server 原生 HA 集群，Web 前端多实例部署，数据库采用主从高可用架构，避免单点故障；
6. **资源限制规范**：所有组件必须配置合理的 CPU / 内存资源请求与限制，避免占用 K8s 节点的过多资源，影响业务应用运行；
7. **网络安全规范**：Zabbix Server 与 Web 前端仅通过 Ingress 对外暴露，且必须配置 HTTPS 加密；Agent 2 与 Server 之间的通信配置 TLS 加密，避免数据泄露；
8. **与云原生生态整合**：结合 Prometheus Exporter 采集 K8s 生态组件指标，通过 Zabbix API 与 K8s 事件中心、运维平台联动，实现故障自动处置。