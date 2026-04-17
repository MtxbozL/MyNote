## 3.3 无 Agent 监控方式

无 Agent 监控是 Zabbix 针对无法部署 Agent 的场景（网络设备、硬件带外管理、封闭系统、Java 应用等）提供的采集方案，无需在被监控端部署任何代理程序，仅通过标准协议即可实现指标采集，核心分为四类标准化采集方式。

### 3.3.1 Simple Check（简单检查）

#### 3.3.1.1 核心定义与原理

Simple Check 是 Zabbix Server/Proxy 原生实现的 **轻量级无代理采集方式** ，所有采集逻辑完全由 Server/Proxy 端执行，无需被监控端提供任何采集服务，仅需网络可达即可完成探测。

工作原理：Server/Proxy 的`Pinger`进程按照监控项配置的周期，直接向目标地址发送 ICMP 报文或 TCP/UDP 连接请求，根据响应结果计算探测指标，无需被监控端做任何配置适配。

#### 3.3.1.2 适用场景与核心 Key

- 核心适用场景：设备存活探测、端口可用性监控、无法部署 Agent 的极简设备（打印机、摄像头、IoT 终端等）监控、广域网链路质量探测。
- 核心内置 Key：

| Key 名称            | 功能说明        | 返回值              |
| ----------------- | ----------- | ---------------- |
| `icmpping`        | ICMP 存活探测   | 1 为可达，0 为不可达     |
| `icmppingloss`    | ICMP 探测丢包率  | 百分比数值，0-100      |
| `icmppingsec`     | ICMP 探测往返延迟 | 浮点数，单位为秒         |
| `net.tcp.port`    | TCP 端口可用性探测 | 1 为端口可连接，0 为不可连接 |
| `net.udp.service` | UDP 服务可用性探测 | 1 为服务响应正常，0 为异常  |

#### 3.3.1.3 配置约束与注意事项

1. Server/Proxy 运行用户需具备 ICMP 报文发送权限（Linux 环境需配置`cap_net_raw`能力），否则 ICMP 探测将失效；
2. 防火墙需放通 Server/Proxy 到目标地址的 ICMP 流量、对应 TCP/UDP 端口的访问权限；
3. 探测并发数由 Server 配置的`StartPingers`参数控制，大规模使用时需调整该参数，避免探测超时；
4. 广域网探测场景需适当调整超时时间，避免网络波动导致的误告警。

### 3.3.2 SNMP 监控

#### 3.3.2.1 核心定义与协议体系

SNMP（Simple Network Management Protocol，简单网络管理协议）是 TCP/IP 协议簇中网络设备管理的 **国际标准协议** ，是 Zabbix 监控交换机、路由器、防火墙、存储、UPS、工业设备等网络资产的核心方式，无需部署 Agent，仅需设备开启 SNMP 服务即可实现全维度指标采集。

SNMP 协议版本分为三类，核心差异与选型规则如下：

|协议版本|认证方式|加密能力|适用场景|
|---|---|---|---|
|SNMPv1|明文团体名|无|老旧设备兼容，严禁用于生产环境|
|SNMPv2c|明文团体名|无|内网可信环境网络设备监控，企业最常用版本|
|SNMPv3|用户名 + 密码认证|支持数据加密与完整性校验|生产环境、公网传输、高安全要求场景，官方推荐版本|

#### 3.3.2.2 核心概念

1. **OID（Object Identifier，对象标识符）**：SNMP 体系中每个监控指标的唯一数字标识，采用树形分级结构，例如`1.3.6.1.2.1.1.1.0`对应系统描述信息，是 SNMP 采集的最小单元。
2. **MIB（Management Information Base，管理信息库）**：SNMP 指标的结构化定义文件，以文本格式描述 OID 对应的指标名称、数据类型、含义、取值范围，实现 OID 的语义化解析，厂商会为自有设备提供专属 MIB 文件。

#### 3.3.2.3 采集实现与配置规范

Zabbix SNMP 采集由 Server/Proxy 的`SNMP Poller`进程处理，核心分为两种采集模式：

1. **单 OID 采集**：通过`snmp.get[<OID>,<community>]`Key 获取单个 OID 对应的指标值，适用于单个固定指标监控；
2. **批量 OID 采集**：通过`snmp.walk[<OID根节点>,<community>]`Key 获取指定 OID 分支下的所有指标，适用于多实例指标采集，与 LLD 低级别发现结合可实现端口、传感器等资源的自动发现与监控。

核心配置要求：

1. 设备端开启 SNMP 服务，配置访问控制列表，仅允许 Zabbix Server/Proxy 的 IP 地址访问，配置对应版本的团体名 / 认证信息；
2. 防火墙放通 Server/Proxy 到设备的 161/UDP 端口（SNMP 默认服务端口）、162/UDP 端口（SNMP Trap 告警端口）；
3. Zabbix 主机配置中填写正确的 SNMP 版本、团体名 / 认证加密参数，监控项类型选择对应 SNMP 版本；
4. 生产环境优先使用厂商提供的官方模板，避免手动配置 OID 导致的指标错误。

#### 3.3.2.4 运维工具

可通过`snmpwalk`、`snmpget`命令行工具提前验证 OID 的有效性与设备 SNMP 配置的正确性，是 SNMP 监控故障排查的核心工具，示例：

```bash
# SNMPv2c 测试OID获取
snmpwalk -v 2c -c public 192.168.1.1 1.3.6.1.2.1.2.2.1.10
# SNMPv3 加密认证测试
snmpget -v 3 -u zabbix -a SHA -A AuthPass123 -x AES -X EncPass123 -l authPriv 192.168.1.1 1.3.6.1.2.1.1.3.0
```

### 3.3.3 JMX 监控

#### 3.3.3.1 核心定义与架构原理

JMX（Java Management Extensions，Java 管理扩展）是 Java 平台的标准管理 API，用于监控和管理 JVM 虚拟机、Java 应用与 Java 中间件（Tomcat、Spring Boot、Dubbo 等）。Zabbix 通过 **Zabbix Java Gateway** 组件实现 JMX 协议的适配与指标采集，是 Java 生态监控的核心无代理方案。

核心架构链路：`Zabbix Server/Proxy → Zabbix Java Gateway → 目标Java应用JMX端口`

- Zabbix Java Gateway：独立的 Java 进程，作为 Server 与 Java 应用之间的协议代理，负责处理 JMX 连接、MBean 数据解析、指标转换，是 JMX 采集的核心中间件；
- 目标 Java 应用：需开启 JMX 远程监控功能，暴露 JMX 服务端口，无需部署任何 Agent 或插件。

#### 3.3.3.2 配置规范与部署流程

1. **Java Gateway 部署**：通过系统包管理器安装`zabbix-java-gateway`包，配置`zabbix_java_gateway.conf`核心参数（监听端口、线程数、超时时间），启动服务并配置开机自启。
2. **Server 端配置**：修改`zabbix_server.conf`，配置`JavaGateway`（Java Gateway 地址）、`JavaGatewayPort`（监听端口）、`StartJavaPollers`（JMX 采集进程数），重启 Zabbix Server 服务生效。
3. **Java 应用端配置**：在 Java 应用启动命令中添加 JVM 参数，开启 JMX 远程监控，核心参数如下：
    
    ```bash
    -Dcom.sun.management.jmxremote
    -Dcom.sun.management.jmxremote.port=12345
    -Dcom.sun.management.jmxremote.authenticate=true
    -Dcom.sun.management.jmxremote.ssl=true
    -Dcom.sun.management.jmxremote.rmi.port=12345
    -Djava.rmi.server.hostname=应用IP地址
    ```
    
4. **Zabbix 监控项配置**：主机配置中填写 JMX 接口 IP 与端口，监控项类型选择 “JMX 代理”，Key 格式为`jmx.get[<MBean名称>,<属性名称>]`，支持复合属性与嵌套属性提取。

#### 3.3.3.3 进阶特性与最佳实践

1. Zabbix Agent 2 原生支持 JMX 采集，无需部署独立的 Java Gateway 组件，简化部署架构，是 7.0 LTS 版本推荐的 JMX 采集方案；
2. 生产环境必须开启 JMX 认证与 SSL 加密，禁止公网暴露 JMX 端口，避免未授权访问与远程代码执行风险；
3. JMX 端口与 RMI 端口需配置为同一端口，简化防火墙规则配置，仅需放通单个 TCP 端口；
4. 优先使用官方提供的 JVM、Tomcat 等标准化模板，避免手动配置 MBean 导致的指标错误。

### 3.3.4 IPMI 监控

#### 3.3.4.1 核心定义与原理

IPMI（Intelligent Platform Management Interface，智能平台管理接口）是硬件级监控的国际标准协议，通过物理服务器的 BMC（Baseboard Management Controller，基板管理控制器）模块实现带外硬件监控，无需操作系统运行，仅需 BMC 模块通电联网即可采集数据，是物理服务器硬件监控的无代理标准方案。

工作原理：Zabbix Server/Proxy 的`IPMI Poller`进程通过 IPMI 协议与服务器 BMC 模块通信，获取硬件传感器的实时数据，完全独立于操作系统，即使服务器关机、操作系统崩溃，仍可正常采集硬件状态数据。

#### 3.3.4.2 适用场景与核心监控维度

- 核心适用场景：数据中心物理服务器的硬件状态监控，覆盖 x86 架构主流服务器厂商（戴尔、惠普、联想、华为等）。
- 核心监控维度：CPU / 内存 / 主板 / 硬盘温度、风扇转速、电源模块状态、供电电压、硬盘健康状态、机箱入侵检测、BMC 固件状态等。

#### 3.3.4.3 配置规范与要求

1. **服务器 BMC 配置**：开启 IPMI 功能，配置 BMC 静态 IP 地址、管理员用户名与密码、IPMI 访问权限，优先使用 IPMI v2.0 版本。
2. **Server/Proxy 环境准备**：安装`OpenIPMI`工具包，修改`zabbix_server.conf`配置`StartIPMIPollers`（IPMI 采集进程数），重启 Zabbix Server 服务生效。
3. **网络与防火墙配置**：放通 Server/Proxy 到 BMC 地址的 623/UDP 端口（IPMI 默认服务端口），生产环境 BMC 需部署在独立带外管理网络，禁止接入业务网络与公网。
4. **Zabbix 配置**：主机配置中填写 IPMI 接口 IP、端口、用户名与密码，监控项类型选择 “IPMI 代理”，Key 格式为`ipmi.get[<传感器ID/名称>]`，可通过 IPMI 自动发现规则实现传感器的批量监控。

#### 3.3.4.4 安全约束

1. BMC 必须配置强密码，定期更换，禁止使用默认密码；
2. 带外管理网络需做严格的访问控制，仅允许 Zabbix Server/Proxy 与运维管理地址访问；
3. 禁止在公网暴露 BMC 端口与 IP 地址，避免未授权访问与服务器控制权泄露。
