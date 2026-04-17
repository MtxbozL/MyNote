
## 9.3 Zabbix Server 核心参数调优

Zabbix Server 的性能调优核心分为**进程调度类参数**与**内存缓存类参数**两大类，所有参数均在`zabbix_server.conf`中配置，修改后需重启 Server 服务生效，调优的核心依据是 Server 内部监控指标。

### 9.3.1 进程调度类参数调优

进程调度类参数控制 Server 各类采集、处理进程的并发数量，核心目标是让各类进程的平均繁忙率稳定在 70% 以下，避免进程阻塞导致的采集延迟、数据丢失。

核心高频参数详解与调优规范如下：

表格

|参数名称|核心职能|调优依据|生产环境配置规范|
|---|---|---|---|
|StartPollers|被动模式采集轮询器进程数，负责被动 Agent、Simple Check 等采集任务|对应内部监控项`zabbix[process,poller,avg,busy]`（Poller 进程平均繁忙率）|初始值 = CPU 核心数 ×2；繁忙率＞70% 时逐步增加，最大不超过 CPU 核心数 ×4；禁止设置过高导致上下文切换开销飙升|
|StartPollersUnreachable|不可达主机轮询器进程数，负责处理不可达主机的探测任务|对应内部监控项`zabbix[process,poller_unreachable,avg,busy]`|初始值 = StartPollers 的 10%-20%；大规模环境建议 8-16 个，避免不可达主机占用正常采集进程资源|
|StartTrappers|Trapper 进程数，负责处理主动模式 Agent 上报、zabbix_sender 推送、Proxy 数据上报|对应内部监控项`zabbix[process,trapper,avg,busy]`|主动模式 / Proxy 大规模场景初始值 = 20-50；繁忙率＞70% 时逐步增加，最大不超过 200；是主动模式架构的核心调优参数|
|StartPingers|ICMP 探测进程数，负责 Simple Check 的 icmpping 系列探测|对应内部监控项`zabbix[process,pinger,avg,busy]`|初始值 = 4-8；大规模网络设备监控场景可调整为 16-32；与`icmpping`类监控项数量成正比|
|StartDiscoverers|网络发现进程数，负责网络发现规则的网段扫描任务|对应内部监控项`zabbix[process,discoverer,avg,busy]`|初始值 = 1-2；多网段大规模发现场景不超过 5；禁止设置过高，避免扫描占用过多资源|
|StartHTTPPollers|HTTP Agent 采集进程数，负责 Web 场景、HTTP Agent 监控项的请求处理|对应内部监控项`zabbix[process,http poller,avg,busy]`|初始值 = 2-4；大规模 API/Web 监控场景可调整为 8-16；与 HTTP 类监控项数量成正比|
|StartJavaPollers|JMX 采集进程数，负责通过 Java Gateway 处理 JMX 监控项|对应内部监控项`zabbix[process,java poller,avg,busy]`|无 JMX 监控时设置为 0；有 JMX 监控时初始值 = 2-4，最大不超过 16|
|StartIPMIPollers|IPMI 采集进程数，负责 IPMI 硬件监控任务|对应内部监控项`zabbix[process,ipmi poller,avg,busy]`|无 IPMI 监控时设置为 0；物理服务器硬件监控场景初始值 = 2-8|

**生产环境硬性规范**：

1. 所有未使用的采集类型，对应进程数必须设置为 0，减少不必要的资源占用；
2. 所有进程的总数量不得超过 CPU 核心数的 10 倍，避免系统负载过高；
3. 进程数调整必须逐步迭代，每次调整幅度不超过 50%，调整后持续观察繁忙率与系统负载，禁止一次性大幅调整。

### 9.3.2 内存缓存类参数调优

缓存类参数控制 Server 各类内存缓存的大小，核心目标是减少磁盘 IO、提升数据处理与触发器计算效率，保证缓存空闲率稳定在 20% 以上，最低不得低于 5%，避免缓存不足导致的性能雪崩。

核心高频参数详解与调优规范如下：

表格

|参数名称|核心职能|调优依据|生产环境配置规范|
|---|---|---|---|
|CacheSize|配置缓存大小，存储主机、监控项、触发器、模板等静态配置数据|对应内部监控项`zabbix[wcache,config,pfree]`（配置缓存空闲百分比）|最小值 128M，默认 32M；小规模环境 256M-512M，大规模环境 1G-4G；空闲率＜20% 时扩容|
|HistoryCacheSize|历史数据缓存大小，缓存待写入数据库的原始采集数据|对应内部监控项`zabbix[wcache,history,pfree]`（历史缓存空闲百分比）|最小值 128M，默认 16M；小规模环境 512M，中大规模环境 1G-8G；空闲率＜20% 时扩容|
|TrendCacheSize|趋势数据缓存大小，缓存待写入数据库的小时级聚合趋势数据|对应内部监控项`zabbix[wcache,trend,pfree]`（趋势缓存空闲百分比）|最小值 128M，默认 4M；小规模环境 256M，中大规模环境 512M-4G；空闲率＜20% 时扩容|
|ValueCacheSize|值缓存大小，缓存触发器表达式计算所需的历史数据，是触发器计算性能的核心参数|对应内部监控项`zabbix[vcache,cache,pfree]`（值缓存空闲百分比）|最小值 128M，默认 8M；小规模环境 512M，大规模环境 2G-16G；触发器告警延迟时优先扩容该参数|
|HistoryIndexCacheSize|历史数据索引缓存大小，优化历史数据查询性能|对应内部监控项`zabbix[hcache,index,pfree]`|最小值 128M，默认 4M；大规模环境 512M-2G；Grafana 查询频繁场景需重点扩容|

**生产环境硬性规范**：

1. 所有缓存参数的总和不得超过系统可用内存的 60%，避免占用操作系统与数据库的内存资源；
2. 大规模环境下，ValueCacheSize 与 HistoryCacheSize 是优先调优的核心参数，直接决定数据处理与告警计算的性能；
3. 缓存扩容后必须持续观察空闲率，禁止过度分配内存导致的系统资源浪费。

### 9.3.3 其他核心性能参数

表格

|参数名称|核心职能|生产环境配置规范|
|---|---|---|
|Timeout|所有采集、请求的超时时间上限|默认 3s，最大不超过 30s；跨网络场景可调整为 5-10s，过长的超时时间会导致进程阻塞|
|HousekeepingFrequency|Housekeeper 管家进程的执行频率，单位小时|大规模环境建议调整为 12-24 小时，仅在业务低峰期执行；使用表分区 / TimescaleDB 方案时可设置为 0，关闭原生 Housekeeper|
|MaxHousekeeperDelete|单次 Housekeeper 执行的最大删除行数|默认 5000，大规模环境建议降低为 1000-2000，减少单次删除的数据库 IO 压力；使用分区方案时无意义|
|StartDBSyncers|数据库同步进程数，负责将缓存数据写入数据库|初始值 = 4，大规模环境可调整为 8-16；与 NVPS 成正比，需匹配数据库的写入性能|
