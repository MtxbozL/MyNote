#### 核心标准答案

K8s 集群的垃圾资源，指的是不再使用的、闲置的资源，包括终止的 Pod、过期的 ReplicaSet、未使用的 ConfigMap/Secret、悬空的 PV/PVC、异常的 Job、镜像缓存等，清理的核心原则是**先识别、后清理，先测试、后生产，避免误删业务资源**，完整的清理方案分为 6 个维度：

1. **清理终止 / 异常 / 闲置的 Pod**
    
    这是最常见的垃圾资源，包括 Evicted、Completed、Error、CrashLoopBackOff 等状态的 Pod，长期闲置的 Pod。
    
    - 识别命令：
        
        bash
        
        运行
        
        ```
        # 查看所有非Running状态的Pod
        kubectl get pods --all-namespaces | grep -v Running
        # 查看所有Evicted状态的Pod（最常见的垃圾Pod）
        kubectl get pods --all-namespaces | grep Evicted
        # 查看已完成的Job生成的Pod
        kubectl get pods --all-namespaces | grep Completed
        ```
        
    - 清理命令：
        
        bash
        
        运行
        
        ```
        # 批量清理所有Evicted状态的Pod
        kubectl get pods --all-namespaces | grep Evicted | awk '{print $1 " " $2}' | xargs -L1 kubectl delete pod -n
        # 批量清理所有Completed状态的Pod
        kubectl get pods --all-namespaces | grep Completed | awk '{print $1 " " $2}' | xargs -L1 kubectl delete pod -n
        # 批量清理所有Error状态的Pod
        kubectl get pods --all-namespaces | grep Error | awk '{print $1 " " $2}' | xargs -L1 kubectl delete pod -n
        ```
        
    - 进阶优化：配置 Pod 的`ttlSecondsAfterFinished`，Job 完成后自动清理 Pod，避免垃圾资源堆积。
    
2. **清理过期的 ReplicaSet、Deployment 历史版本**
    
    Deployment 更新后，旧的 ReplicaSet 会被保留，用于回滚，长期不清理会堆积大量无用的 ReplicaSet。
    
    - 识别与清理：
        
        bash
        
        运行
        
        ```
        # 查看所有命名空间的ReplicaSet
        kubectl get replicasets --all-namespaces
        # 自动清理：通过Deployment的revisionHistoryLimit参数，限制保留的历史版本数，默认10个，多余的会自动清理
        # 手动清理无关联的ReplicaSet（副本数为0的旧版本）
        kubectl delete replicasets --all-namespaces --field-selector spec.replicas=0
        ```
        
    
3. **清理未使用的 ConfigMap、Secret**
    
    大量创建后未被 Pod、Deployment 引用的 ConfigMap 和 Secret，长期堆积会占用集群资源，增加 apiserver 压力。
    
    - 识别与清理：
        
        bash
        
        运行
        
        ```
        # 工具推荐：使用kube-janitor、kubectl-delete-unused-configmaps等开源工具，自动识别未被引用的ConfigMap/Secret
        # 手动识别：查看ConfigMap是否被Pod挂载、环境变量引用
        # 手动清理指定的闲置ConfigMap/Secret
        kubectl delete configmap <configmap名称> -n <命名空间>
        kubectl delete secret <secret名称> -n <命名空间>
        ```
        
    
4. **清理悬空 / 未使用的 PV、PVC、StorageClass 资源**
    
    PVC 被删除后对应的 PV、未被 PVC 绑定的 PV、已释放但未清理的 PV，都是常见的垃圾资源。
    
    - 识别与清理：
        
        bash
        
        运行
        
        ```
        # 查看PV的状态，Released、Failed状态的PV都是闲置的
        kubectl get pv | grep -v Bound
        # 清理Released状态的PV
        kubectl delete pv $(kubectl get pv | grep Released | awk '{print $1}')
        # 查看未被Pod挂载的PVC
        kubectl get pvc --all-namespaces | grep -v Bound
        # 清理闲置的PVC
        kubectl delete pvc <pvc名称> -n <命名空间>
        ```
        
    
5. **清理异常 / 完成的 Job、CronJob，悬空的 Ingress、Service**
    
    - 识别与清理：
        
        bash
        
        运行
        
        ```
        # 清理已完成/失败的Job
        kubectl delete jobs --all-namespaces --field-selector status.successful=1
        kubectl delete jobs --all-namespaces --field-selector status.failed=1
        # 清理无后端Endpoint的Service（悬空Service）
        kubectl get endpoints --all-namespaces | grep -v "1/" | grep -v "2/" | awk '{print $1 " " $2}' | xargs -L1 kubectl delete service -n
        # 清理无后端Service的Ingress
        kubectl get ingress --all-namespaces | grep -v <正常的Ingress> | 手动筛选后删除
        ```
        
    
6. **清理节点上的镜像缓存、容器日志、临时文件**
    
    集群节点上的闲置镜像、容器日志、临时文件，会占用大量的磁盘空间，需要定期清理。
    
    - 核心清理命令（在每个节点上执行，或通过 DaemonSet 批量执行）：
        
        bash
        
        运行
        
        ```
        # 清理Docker/K8s的闲置镜像、停止的容器、无用的卷、网络
        docker system prune -af
        # 仅清理未被使用的镜像（保留最近使用的）
        docker image prune -af --filter "until=720h"
        # 清理容器日志（按大小/时间清理）
        find /var/lib/docker/containers/ -name "*.log" -type f -mtime +7 -delete
        # 清理K8s的临时目录、闲置卷
        find /var/lib/kubelet/pods/ -type f -mtime +7 -delete
        ```
        
    

#### 专家级拓展

- 生产环境最佳实践：
    
    1. **自动化清理优先**：不要手动清理，优先通过配置自动清理规则，比如 Deployment 的`revisionHistoryLimit`、Job 的`ttlSecondsAfterFinished`、容器日志的轮转策略，从源头避免垃圾资源堆积；
    2. **工具化清理**：使用开源工具实现自动化的识别和清理，比如：
    
    - kube-janitor：自动清理过期的、未使用的 K8s 资源，支持自定义规则；
    - kube-cleanup-operator：自动清理完成的 Job、旧的 ReplicaSet、闲置 Pod；
    - trivy/kyverno：扫描集群中的闲置资源、风险资源，提前预警；
    
    3. **先干跑，后执行**：清理前先通过`--dry-run`参数模拟执行，确认要清理的资源无误后，再正式执行，避免误删业务资源；
    4. **定期执行**：通过 CronJob 在集群内定期执行清理任务，比如每周凌晨低峰期执行，避免垃圾资源长期堆积；
    5. **命名空间隔离**：测试环境、生产环境分开命名空间，测试环境的资源设置过期时间，自动清理。
    
- 安全红线：
    
    1. 严禁在生产环境直接执行批量删除命令，必须先筛选、确认资源无误后，再执行清理；
    2. 生产环境禁止使用`--all`参数批量删除资源，必须精准筛选；
    3. 清理 PV/PVC、ConfigMap/Secret 等持久化资源前，必须确认没有业务使用，避免数据丢失。
    

#### 面试避坑指南

- 严禁只说零散的删除命令，不说「先识别、后清理、自动化优先」的核心原则，会被面试官判定为只会暴力删除，无生产环境集群运维经验；
- 避免只清理 Pod，不说 ReplicaSet、PV/PVC、ConfigMap 等其他垃圾资源，体现你对 K8s 资源的全面理解；
- 不要遗漏自动化清理工具和最佳实践，这是区分普通运维和高级运维的核心点，提到会大幅加分。