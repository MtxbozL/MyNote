
`rule_files`模块用于指定 Prometheus 加载的规则文件路径，规则文件包含两类核心规则：

1. **记录规则（Recording Rule）**：对高频查询的 PromQL 表达式进行预聚合计算，将结果存储为新的时序数据，大幅提升高频查询、大盘渲染的性能；
2. **告警规则（Alerting Rule）**：定义告警触发的 PromQL 表达式、阈值、持续时间与标签、注释，是 Prometheus 告警体系的核心。

### 2.4.1 路径配置规范

`rule_files`为字符串数组类型，支持绝对路径、相对路径与通配符匹配，生产环境推荐使用绝对路径，避免相对路径的歧义问题。

#### 标准配置示例

```yaml
rule_files:
  # 匹配指定目录下的所有yml/yaml文件
  - "/etc/prometheus/rules/*.yml"
  - "/etc/prometheus/rules/*.yaml"
  # 递归匹配子目录下的所有规则文件
  - "/etc/prometheus/rules/**/*.yml"
  # 单独指定单个规则文件
  - "/etc/prometheus/alert_rules/node_alerts.yml"
```

### 2.4.2 规则文件加载机制

1. **加载时机**：Prometheus 进程启动时、热加载触发时，会自动扫描并加载`rule_files`指定的所有规则文件；
2. **合法性校验**：加载前会对规则文件进行语法与语义校验，若任意规则文件校验失败，将终止加载流程，原有已加载的规则继续生效，不会导致服务不可用；
3. **校验工具**：Prometheus 提供`promtool`命令行工具，可提前校验规则文件的合法性，是生产环境变更前的必备步骤，命令如下：

    ```bash
    # 校验单个规则文件
    promtool check rules /etc/prometheus/rules/node_alerts.yml
    # 批量校验目录下所有规则文件
    promtool check rules /etc/prometheus/rules/*.yml
    ```
    

> 注：规则文件的详细语法、记录规则与告警规则的编写规范，将在第七章《Alertmanager 告警路由与通知》中详细讲解，本章仅聚焦规则文件的加载机制。

---
