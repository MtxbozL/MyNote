#### 核心标准答案

我会遵循「从基础到复杂、从本地到集群、从 Pod 到网络组件」的分层排查思路，逐步缩小范围，定位根因，完整流程如下：

##### 第一步：基础信息确认，缩小排查范围

1. 执行`kubectl get pod -o wide`，获取两个 Pod 的 IP 地址、所在节点、所属 Namespace，确认两个 Pod 都处于 Running 状态，没有重启、报错。
2. 确认两个 Pod 的状态正常：执行`kubectl describe pod pod名称`，查看 Pod 的事件，没有 OOM killed、镜像拉取失败、调度失败等异常；执行`kubectl logs pod名称`，查看容器日志，确认应用程序正常启动，没有端口监听失败、服务崩溃等问题。

##### 第二步：基础连通性测试，定位故障层级

1. **Pod 内网络基础测试**：分别进入两个 Pod，执行`ip addr`，确认 Pod 的 IP 地址正确，网卡正常；执行`ping 127.0.0.1`，确认 Pod 内的网络协议栈正常。
2. **同节点 / 跨节点区分测试**：
    
    - 如果两个 Pod 在**同一个节点**：直接在 Pod 内互相 ping 对方的 IP，不通则排查 Pod 的网络命名空间、veth pair、节点上的 iptables 规则、网络插件的本地规则。
    - 如果两个 Pod 在**不同节点**：先在两个节点上互相 ping 对方的节点 IP，确认节点之间网络互通；再在节点上 ping 对方节点上的 Pod IP，确认节点到 Pod 的网络互通；如果节点间互通，但 Pod 间不通，排查节点间的网络隧道、防火墙、安全组、网络插件的跨节点路由。
    

##### 第三步：核心规则与策略排查，定位拦截原因

1. **排查网络策略 NetworkPolicy**：执行`kubectl get networkpolicy -n 对应Namespace`，查看是否有网络策略限制了两个 Pod 之间的访问，比如 Ingress/Egress 规则禁止了对应 Namespace、Pod IP 的访问，这是 Pod 网络不通的高频原因。
2. **排查节点防火墙与安全组**：查看节点的 iptables、firewalld 规则，是否拦截了 Pod 的网段（比如 10.244.0.0/16）；如果是云厂商集群，查看节点的安全组规则，是否放行了集群的 Pod 网段、容器网络端口（比如 Calico 的 BGP 端口 179）。
3. **排查 Service 与域名解析问题**：如果是通过 Service 域名访问不通，先直接 ping Pod IP，Pod IP 通但 Service 不通，排查 Service 的 Endpoints 是否正确（`kubectl get endpoints service名称`），确认 Endpoints 中有对应的 Pod IP 和端口；再排查 kube-proxy 是否正常运行，iptables/ipvs 规则是否正确生成；最后排查 CoreDNS 是否正常，在 Pod 内执行`nslookup service名称`，确认域名解析正常。

##### 第四步：集群网络组件排查，定位底层故障

1. 查看集群网络插件（Calico/Flannel/Cilium）的 Pod 是否正常运行，执行`kubectl get pod -n kube-system | grep 网络插件名称`，确认所有节点上的网络插件 Pod 都处于 Running 状态，没有报错。
2. 查看网络插件的日志，排查是否有路由配置失败、隧道建立失败、iptables 规则写入失败等异常。
3. 查看 CoreDNS 的 Pod 是否正常运行，日志是否有报错，确认集群的域名解析服务正常。

##### 第五步：根因修复与验证

根据定位的根因，针对性处理，比如删除限制访问的网络策略、放通防火墙 / 安全组规则、修复异常的网络插件、重启异常的 CoreDNS，处理完成后，再次在 Pod 内互相 ping、访问对应端口，确认网络恢复正常。

#### 专家级拓展

- 高频故障根因 Top3：
    
    1. 网络策略 NetworkPolicy 配置错误，禁止了 Pod 之间的访问，占比超过 50%。
    2. 节点防火墙 / 安全组没有放通 Pod 网段、容器网络端口，导致跨节点 Pod 网络不通。
    3. 网络插件异常，比如 Calico 的 BGP 邻居建立失败、Flannel 的隧道配置错误，导致跨节点网络不通。
    
- 进阶排查工具：
    
    1. `kubectl debug`：用于调试异常 Pod，给 Pod 附加调试容器，安装 tcpdump、curl 等工具，抓包分析 Pod 的网络请求。
    2. `tcpdump`：在 Pod 的 veth 网卡、节点的物理网卡上抓包，确认数据包是否发出、是否被拦截、是否有回包，精准定位数据包丢失的位置。
    3. `cilium hubble`/`calicoctl`：对应网络插件的专用工具，可查看网络策略的命中情况、路由规则、流量日志，快速定位网络拦截问题。
    

#### 面试避坑指南

- 严禁一上来就排查网络插件、CoreDNS，必须先从最基础的 Pod 状态、IP 连通性开始，逐步深入，体现专业的排查逻辑；
- 避免遗漏网络策略 NetworkPolicy 的排查，这是面试中最常被追问的点，也是生产环境最高频的故障原因；
- 不要只说泛泛的流程，不说同节点 / 跨节点的区分排查，无法体现实际的故障排查经验。