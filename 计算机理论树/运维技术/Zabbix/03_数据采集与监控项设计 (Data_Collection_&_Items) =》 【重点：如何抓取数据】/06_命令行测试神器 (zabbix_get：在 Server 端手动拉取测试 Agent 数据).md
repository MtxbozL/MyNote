
## 3.7 zabbix_get 命令行工具

### 3.7.1 核心定义与作用

`zabbix_get`是 Zabbix 官方提供的、运行在 Server/Proxy 端的命令行测试工具，用于手动向 Zabbix Agent 发起采集请求，获取指定 Key 的监控数据，是 Agent 采集故障排查、Key 有效性验证、自定义脚本测试的核心运维工具，是 Zabbix 采集体系的必备调试工具。

核心作用：

1. 验证 Server/Proxy 与 Agent 之间的网络连通性与端口可用性；
2. 验证 Agent 内置 Key 的有效性与返回值是否符合预期；
3. 测试 UserParameter 自定义 Key 的脚本执行结果，排查自定义采集故障；
4. 定位采集失败的根因：区分网络问题、Agent 配置问题、权限问题、Key 语法问题。

### 3.7.2 核心语法与参数

标准命令格式：

bash

运行

```
zabbix_get -s <Agent IP地址> -p <Agent端口> -k <监控项Key>
```

核心高频参数：

表格

|参数|全称|功能说明|
|---|---|---|
|-s|--host|目标 Agent 的 IP 地址或主机名，必填|
|-p|--port|Agent 的监听端口，默认 10050|
|-k|--key|待测试的监控项 Key，必填|
|-I|--source-address|指定源 IP 地址，适用于 Server 多网卡场景|
|-t|--timeout|超时时间，单位秒，默认 30s|
|-v|--verbose|详细输出模式，用于深度故障排查|

### 3.7.3 常用示例

1. 测试 Agent 内置 CPU 利用率 Key：
    
    bash
    
    运行
    
    ```
    zabbix_get -s 192.168.1.100 -k "system.cpu.util[,user]"
    ```
    
2. 测试自定义 UserParameter Key：
    
    bash
    
    运行
    
    ```
    zabbix_get -s 192.168.1.100 -k "mysql.threads.connected"
    ```
    
3. 带参数的 Key 测试：
    
    bash
    
    运行
    
    ```
    zabbix_get -s 192.168.1.100 -k "vfs.fs.size[/,pused]"
    ```
    

### 3.7.4 使用规则与故障排查

1. 必须在 Zabbix Server/Proxy 端运行，Agent 的`Server`配置参数仅允许 Server/Proxy 的 IP 地址访问，其他主机运行会被 Agent 拒绝；
2. 若返回`ZBX_NOTSUPPORTED`，说明 Key 不存在、参数错误、脚本执行失败、权限不足，需针对性排查 Agent 配置与脚本；
3. 若返回连接超时，说明网络不通、防火墙拦截、Agent 服务未启动、端口配置错误；
4. 若返回权限拒绝，说明 Agent 的`Server`配置未包含当前主机的 IP 地址，需修改 Agent 配置并重启服务；
5. 生产环境排查采集故障时，优先使用`zabbix_get`工具验证链路有效性，缩小故障范围。

## 3.8 采集方式选型矩阵

为便于生产环境选型，本章所有采集方式的核心特性与适用场景汇总如下：

表格

|采集方式|核心依赖|部署复杂度|核心优势|核心限制|首选适用场景|
|---|---|---|---|---|---|
|Zabbix Agent|被控端部署 Agent|低|精细化、全维度、低资源占用|需在被控端部署代理|服务器操作系统、中间件监控|
|Simple Check|仅需网络可达|极低|无需被控端配置、轻量|仅支持存活与端口探测|设备存活监控、极简设备可用性探测|
|SNMP|设备支持 SNMP 协议|低|网络设备标准协议、无代理|仅支持 SNMP 兼容设备|交换机、路由器、防火墙、存储设备监控|
|JMX|Java 应用开启 JMX|中|Java 生态原生标准、无代理|仅支持 Java 应用|JVM、Tomcat、Spring Boot 等 Java 应用监控|
|IPMI|服务器 BMC 支持 IPMI|低|硬件级带外监控、无关操作系统|仅支持物理服务器带外管理|物理服务器硬件状态监控|
|Zabbix Trapper|zabbix_sender 工具|中|主动推送、异步事件驱动、非周期采集|需预先创建 Trapper 监控项|计划任务结果、非周期数据、单向网络场景|
|Calculated Items|已有监控项数据|极低|原生衍生指标、无额外依赖|依赖已有采集数据|复合指标、聚合指标、业务成功率计算|
|HTTP Agent|目标提供 HTTP 接口|低|原生 API 采集、无脚本依赖、全场景适配|仅支持 HTTP/HTTPS 协议|API 接口、Web 服务、云服务、IoT 设备监控|

继续下一章