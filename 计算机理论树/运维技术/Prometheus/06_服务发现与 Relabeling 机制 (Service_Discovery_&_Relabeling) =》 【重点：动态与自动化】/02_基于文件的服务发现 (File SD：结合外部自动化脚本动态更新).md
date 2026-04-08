
File SD 是 Prometheus 内置的、兼容性最强的动态服务发现机制，是连接 Prometheus 与外部自动化系统的通用桥梁，也是所有动态服务发现中最易上手、适配性最广的方案。

### 6.2.1 核心工作原理

File SD 通过定期读取本地指定的 JSON/YAML 文件，自动更新抓取目标列表，无需触发 Prometheus 热加载，完全无服务中断。核心执行流程如下：

1. Prometheus 启动时，加载`file_sd_configs`中指定的目标文件，解析抓取目标与标签；
2. 按照配置的刷新间隔（默认 5 分钟），定期重新读取文件内容，检测目标变更；
3. 若文件内容发生变化，自动增量更新抓取目标，无需重启或热加载；
4. 支持通配符匹配多个文件，自动监听目录内的文件变更。

### 6.2.2 标准配置规范

#### 1. 目标文件格式

支持 YAML 与 JSON 两种格式，核心结构与`static_configs`完全一致，便于从静态配置平滑迁移，每个文件包含一个或多个目标组，每个目标组由`targets`（地址列表）与`labels`（附加标签）构成。

标准 YAML 目标文件示例（`/etc/prometheus/sd_configs/node_targets.yml`）：

```yaml
- targets:
  - 192.168.1.10:9100
  - 192.168.1.11:9100
  labels:
    env: prod
    idc: guangzhou-az1
    business: infrastructure
- targets:
  - 192.168.1.20:9104
  labels:
    env: prod
    idc: guangzhou-az1
    business: database
```

#### 2. Prometheus 侧配置

在`scrape_configs`中通过`file_sd_configs`指定目标文件路径与刷新间隔，标准配置如下：

```yaml
scrape_configs:
  - job_name: "file_sd_node"
    scrape_interval: 15s
    file_sd_configs:
      - files:
          - /etc/prometheus/sd_configs/node_targets.yml
          - /etc/prometheus/sd_configs/*.yml # 通配符匹配目录下所有yml文件
        refresh_interval: 30s # 刷新间隔，生产环境推荐30s-1m，默认5m
```

### 6.2.3 适用场景与自动化集成

1. **通用适配场景**：无内置服务发现支持的传统物理机 / 虚拟机集群、自研服务注册体系、混合云环境，是所有环境的兜底方案；
2. **自动化集成**：可通过 Ansible、SaltStack、CMDB 同步脚本、CI/CD 流水线自动生成 / 更新目标文件，实现监控目标与服务生命周期的全自动化同步；
3. **灰度变更场景**：可通过修改文件内容实现抓取目标的灰度新增 / 下线，无任何服务中断风险，适配生产环境的变更管控要求；
4. **多环境统一管理**：通过不同的目标文件拆分开发、测试、生产环境的监控目标，实现配置的标准化与隔离。

### 6.2.4 生产环境最佳实践

1. 目标文件按业务域、环境、Job 类型拆分，避免单文件过大，便于权限管理与维护；
2. 刷新间隔设置合理，避免过短导致磁盘 IO 压力过大，过长导致目标更新不及时，生产环境推荐 30s-1m；
3. 目标文件生成后，需通过`promtool check config`命令校验配置合法性，避免格式错误导致服务发现失效；
4. 为目标文件设置只读权限，仅允许自动化脚本修改，禁止手动直接修改，避免人为误操作。

---
