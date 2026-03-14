#### 核心标准答案

1. **GPU 监控三剑客**
    
    行业通用的 GPU 监控核心工具组合（三剑客）为：**NVIDIA-SMI、NVIDIA DCGM、Prometheus+Grafana**，三者配合实现 GPU 指标的采集、存储、可视化、告警，覆盖从单机到集群的全场景 GPU 监控。
    
    - NVIDIA-SMI：NVIDIA 官方自带的 GPU 管理命令行工具，是 GPU 监控的基础，可实时查看单节点 GPU 的利用率、显存使用率、温度、功耗、ECC 错误等所有核心指标，无需额外安装；
    - NVIDIA DCGM（Data Center GPU Manager）：NVIDIA 官方针对数据中心级 GPU 集群推出的监控管理工具，提供更丰富的指标采集、健康检查、故障诊断、性能调优能力，支持多 GPU、多节点的集群级监控，是生产环境 GPU 集群监控的核心组件；
    - Prometheus+Grafana：时序数据库 + 可视化工具，通过 DCGM Exporter 采集 GPU 指标，存储到 Prometheus，通过 Grafana 实现 GPU 指标的可视化看板、聚合计算、告警配置，是集群级 GPU 监控的标准方案。
        
        补充：轻量级单机监控场景，也有「NVIDIA-SMI、nvtop、gpustat」的三剑客说法，生产环境集群监控，以前者为准。
    
2. **GPU 利用率求和的实现方式**
    
    按场景分为单机单卡、单机多卡、集群多节点三种维度，实现方式如下：
    
    - **场景 1：单机多 GPU 卡的利用率求和（命令行方式）**
        
        基于 NVIDIA-SMI 命令，通过 awk 实现求和计算，核心命令：
        
        bash
        
        运行
        
        ```
        # 实时计算单机所有GPU的利用率之和
        nvidia-smi --query-gpu=utilization.gpu --format=csv,noheader,nounits | awk '{sum += $1} END {print "GPU总利用率之和："sum"%"}'
        
        # 计算单机所有GPU的平均利用率
        nvidia-smi --query-gpu=utilization.gpu --format=csv,noheader,nounits | awk '{sum += $1; count++} END {print "GPU平均利用率："sum/count"%"}'
        ```
        
    - **场景 2：集群级 GPU 利用率求和（监控系统方式，生产环境标准方案）**
        
        基于 Prometheus+DCGM Exporter，通过 PromQL 实现多节点、多 GPU 的利用率求和、聚合计算，核心实现：
        
        1. 部署 DCGM Exporter 到 GPU 节点，采集 GPU 指标，核心指标为`DCGM_FI_DEV_GPU_UTIL`（GPU 核心利用率）；
        2. Prometheus 配置抓取 DCGM Exporter 的指标，实现全集群 GPU 指标的集中存储；
        3. 通过 PromQL 实现求和计算：
            
            promql
            
            ```
            # 全集群所有GPU的利用率总和
            sum(DCGM_FI_DEV_GPU_UTIL)
            
            # 按节点分组，计算每个节点的GPU利用率总和
            sum by (instance) (DCGM_FI_DEV_GPU_UTIL)
            
            # 按业务线/命名空间分组，计算对应GPU集群的利用率总和
            sum by (namespace) (DCGM_FI_DEV_GPU_UTIL)
            
            # 全集群GPU的平均利用率
            avg(DCGM_FI_DEV_GPU_UTIL)
            ```
            
        4. 在 Grafana 中配置 PromQL 查询，实现 GPU 利用率总和的可视化看板、趋势图、阈值告警。
        
    - **场景 3：代码层面实现 GPU 利用率求和**
        
        基于 Python 的 pynvml 库（NVIDIA 官方 Python 绑定库），在代码中实现 GPU 利用率的采集、求和、统计，适配自定义监控、业务埋点场景。
    

#### 专家级拓展

- GPU 监控的核心指标（生产环境必监控）：
    
    1. 计算指标：GPU 核心利用率、SM 流多处理器利用率、张量核心利用率；
    2. 显存指标：显存使用率、显存已用 / 总量、显存带宽利用率；
    3. 硬件健康指标：GPU 温度、功耗、风扇转速、电源状态；
    4. 错误与故障指标：ECC 错误、PCIe 错误、GPU 掉卡、进程崩溃、XID 错误；
    5. 业务指标：GPU 上运行的进程、显存占用、任务执行时长、任务排队数。
    
- 生产环境最佳实践：
    
    1. 大规模 GPU 集群，必须使用 DCGM 替代单纯的 nvidia-smi，DCGM 支持更细粒度的指标采集、GPU 健康诊断、故障隔离，适配数据中心级的 GPU 集群管理；
    2. 告警配置不仅要配置单 GPU 的利用率阈值，还要配置集群整体的 GPU 利用率总和、平均利用率，实现集群资源的整体调度与容量规划；
    3. 配合 K8s 调度，通过 GPU 利用率的聚合指标，实现 GPU 任务的智能调度、弹性扩缩容，提升 GPU 集群的资源利用率。
    

#### 面试避坑指南

- 严禁说错 GPU 监控三剑客，避免把无关工具纳入，必须以 NVIDIA 官方的标准工具为准，这是面试的基础得分点；
- 避免只说单机单卡的利用率查看，不说集群级的聚合求和实现，面试问这个问题，核心是考察你有没有大规模 GPU 集群的监控运维经验；
- 不要只说求和命令，不说生产环境的 Prometheus+Grafana 标准方案，会被判定为只会基础命令，无实际生产环境监控落地能力。