#### 核心标准答案

K8s 中应用的滚动更新与回滚，核心通过**Deployment 资源**实现，Deployment 通过管理底层的 ReplicaSet，实现无感知的滚动更新和版本回滚，保证更新过程中业务不中断。

##### 一、滚动更新的实现原理与流程

1. **核心原理**
    
    滚动更新的核心是「**增量替换**」：不一次性停止所有旧版本 Pod，而是逐步创建新版本 Pod，等新版本 Pod 就绪后，再逐步删除旧版本 Pod，整个过程中始终有可用的服务实例，保证业务零中断。
    
    核心控制参数：
    
    - `maxSurge`：滚动更新过程中，最多可以超出期望副本数的比例 / 数量，默认 25%，控制新增的新版本 Pod 的速度；
    - `maxUnavailable`：滚动更新过程中，最多可以不可用的 Pod 比例 / 数量，默认 25%，控制业务的最低可用实例数。
    
2. **完整执行流程**
    
    1. 用户修改 Deployment 的 Pod 模板（比如更新镜像版本、环境变量、配置），触发滚动更新；
    2. Deployment 创建一个新的 ReplicaSet，用于管理新版本的 Pod；
    3. Deployment 根据`maxSurge`配置，逐步创建新版本的 Pod，直到新版本 Pod 数量达到`maxSurge`允许的上限；
    4. 等待新版本 Pod 通过就绪探针（readinessProbe）检测，确认服务正常就绪；
    5. 就绪的新版本 Pod 加入 Service 的 Endpoints 列表，开始接收流量；
    6. Deployment 根据`maxUnavailable`配置，逐步删除旧版本的 Pod，同时继续创建剩余的新版本 Pod；
    7. 重复上述步骤，直到所有 Pod 都替换为新版本，旧的 ReplicaSet 保留（用于回滚），滚动更新完成。
    

##### 二、版本回滚的实现原理与流程

1. **核心原理**
    
    Deployment 会保留每次滚动更新的历史版本（对应旧的 ReplicaSet），当新版本出现问题时，可快速回滚到之前的历史版本，回滚的本质是将 Deployment 的 Pod 模板恢复到历史版本，然后按照滚动更新的逻辑，将 Pod 从新版本替换为旧版本。
2. **核心操作与流程**
    
    1. 查看更新历史：执行`kubectl rollout history deployment/[deployment名称]`，查看所有的历史版本，确认要回滚的版本号；
    2. 查看版本详情：执行`kubectl rollout history deployment/[deployment名称] --revision=版本号`，查看对应版本的配置，确认无误；
    3. 执行回滚：执行`kubectl rollout undo deployment/[deployment名称] --to-revision=版本号`，回滚到指定的历史版本；如果不加`--to-revision`，默认回滚到上一个版本；
    4. 验证回滚：执行`kubectl get deployment`、`kubectl get pods`，确认 Pod 已替换为旧版本，服务正常运行。
    

#### 专家级拓展

- 生产环境最佳实践：
    
    1. 必须配置就绪探针和存活探针：滚动更新过程中，只有就绪探针检测通过的 Pod 才会接收流量，避免未就绪的 Pod 接收请求导致业务报错，这是滚动更新零中断的核心保障；
    2. 合理调整滚动策略：核心业务调小`maxSurge`和`maxUnavailable`（比如 10%），保证更新过程中足够多的可用实例；非核心业务可调大参数，加快更新速度；
    3. 版本历史保留：通过`revisionHistoryLimit`配置保留的历史版本数（默认 10 个），避免过多的旧 ReplicaSet 占用集群资源，同时保留足够的版本用于回滚；
    4. 灰度发布 / 金丝雀发布：基于 Deployment 可实现简单的灰度发布，比如先更新少量实例，验证无误后再全量更新；复杂的灰度发布可通过 Flagger、Argo Rollouts 等工具实现。
    
- 进阶知识点：
    
    - 触发滚动更新的条件：只有修改 Deployment 的 Pod 模板（spec.template）才会触发滚动更新，修改副本数、标签等不会触发；
    - 暂停与恢复更新：可通过`kubectl rollout pause`暂停滚动更新，验证新版本无误后，再通过`kubectl rollout resume`恢复更新，适合灰度验证场景；
    - 回滚的限制：如果旧的 ReplicaSet 被删除了，就无法回滚到对应的历史版本，因此不要手动删除 Deployment 的旧 ReplicaSet。
    

#### 面试避坑指南

- 严禁只说更新和回滚的命令，不说底层原理和 ReplicaSet 的作用，会被面试官判定为「只会敲命令，不懂底层实现」；
- 避免遗漏就绪探针的作用，这是生产环境滚动更新不中断业务的核心，不提会被认为无实际生产环境部署经验；
- 不要混淆滚动更新和蓝绿发布，滚动更新是增量替换，资源占用少；蓝绿发布是一次性创建全部新版本 Pod，验证通过后一次性切换流量，资源占用高，二者不能混为一谈。