#### 核心标准答案

K8s 中 Pod 的访问分为**集群内访问**和**集群外访问**两种场景，核心流程完全不同，具体如下：

##### 一、集群内客户端访问 Pod（最常见场景）

核心是通过 Service 资源作为稳定的访问入口，完整流程：

1. **域名解析**：客户端 Pod 通过 Service 名称访问目标服务，首先请求集群内的 CoreDNS 服务，CoreDNS 将 Service 名称解析为对应的 ClusterIP（Service 的集群内虚拟 IP）。
2. **流量转发到 Service**：客户端 Pod 发起请求，目标地址是 ClusterIP，数据包经过节点的内核网络栈，被 kube-proxy 组件拦截。
3. **kube-proxy 负载均衡**：kube-proxy 根据 Service 对应的 Endpoints 列表（关联的健康 Pod 的 IP + 端口），通过 iptables/ipvs 规则，将数据包转发到对应的后端 Pod，同时做 DNAT 地址转换，将目标 IP 从 ClusterIP 改为目标 Pod 的 IP。
4. **数据包到达 Pod**：数据包通过 CNI 网络插件（Calico/Flannel）的网络隧道 / 路由，到达目标 Pod 所在的节点，再转发到目标 Pod 的网络命名空间，最终被 Pod 内的容器接收。
5. **响应返回**：Pod 处理完请求后，响应数据包按原路径返回客户端 Pod，完成整个访问流程。

##### 二、集群外客户端访问 Pod

集群外无法直接访问 Pod 的内网 IP，必须通过 K8s 提供的入口方式访问，主流有 3 种方式，流程分别如下：

1. **NodePort 方式**
    
    - Service 的类型设置为 NodePort，会在集群的每个节点上开放一个固定的端口（默认 30000-32767）；
    - 集群外客户端通过「节点的公网 IP+NodePort 端口」发起请求；
    - 数据包到达节点后，被 kube-proxy 拦截，转发到对应的后端 Pod，流程和集群内访问一致。
    
2. **LoadBalancer 方式**
    
    - 仅适用于云厂商的 K8s 集群，Service 类型设置为 LoadBalancer，云厂商会自动分配一个公网负载均衡实例和公网 IP；
    - 集群外客户端通过公网负载均衡 IP 访问，流量先到达云厂商的负载均衡器，再转发到集群节点的 NodePort，最终通过 kube-proxy 转发到后端 Pod。
    
3. **Ingress 方式（生产环境首选）**
    
    - 集群内部署 Ingress Controller（比如 Ingress-Nginx），作为集群统一的流量入口，通过 NodePort/LoadBalancer 暴露到公网；
    - 配置 Ingress 规则，定义域名、URL 路径与后端 Service 的映射关系；
    - 集群外客户端通过域名访问，请求先到达 Ingress Controller，Ingress Controller 根据域名和路径，将请求转发到对应的 Service，再由 Service 转发到后端 Pod。
    

#### 专家级拓展

- 核心底层细节：
    
    1. kube-proxy 的两种模式：iptables 模式是默认模式，通过内核 iptables 规则实现转发，适合中小规模集群；ipvs 模式基于内核 IPVS 模块，性能更高，支持更多负载均衡算法，适合大规模集群、高并发场景。
    2. ClusterIP 是虚拟 IP，没有对应的网络设备，仅存在于 iptables/ipvs 规则中，仅在集群内可路由，集群外无法直接访问。
    3. Endpoints 资源：Service 和 Pod 之间通过 Endpoints 关联，Endpoints 会实时更新健康的 Pod 的 IP 和端口，kube-proxy 根据 Endpoints 更新转发规则，保证只会转发到健康的 Pod。
    
- 生产环境最佳实践：集群外访问优先使用 Ingress 方式，统一管理域名、SSL 证书、限流、熔断等能力，避免大量 NodePort/LoadBalancer 占用端口和公网资源；核心业务通过 LoadBalancer+Ingress 结合的方式，保证高可用。

#### 面试避坑指南

- 严禁混淆集群内和集群外的访问流程，不说 CoreDNS 和 Service 的核心作用，会被面试官判定为对 K8s 网络模型理解不透彻；
- 避免只说访问方式，不说完整的数据包流转流程，面试问这个问题，核心是考察你对 K8s 网络底层原理的理解；
- 不要遗漏 kube-proxy 和 CNI 网络插件的作用，这是 K8s 网络转发的核心组件，不提会被认为理解不全面。