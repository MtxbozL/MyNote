## 3.4 Zabbix Trapper 与 zabbix_sender

### 3.4.1 核心定义与原理

Zabbix Trapper（陷阱器）是 Zabbix 的 **事件驱动型主动推送采集模式** ，与 Agent 周期轮询采集不同，Trapper 允许被监控端在任意时间主动向 Zabbix Server/Proxy 推送监控数据，无需等待 Server 的采集调度，是异步、非周期型数据采集的核心方案。

工作原理：

1. Zabbix Server 启动`Trapper`进程（由`StartTrappers`参数控制），监听 10051/TCP 端口，接收客户端推送的数据；
2. 被监控端通过`zabbix_sender`工具或自定义程序，将主机名、监控项 Key、指标值、时间戳推送给 Server；
3. Server 验证主机与 Trapper 类型监控项的有效性，验证通过后将数据直接写入数据库，无需经过轮询调度流程。

### 3.4.2 核心适用场景

1. 非周期型数据上报：计划任务执行结果、备份任务状态、一次性业务事件结果、脚本执行返回值等非固定周期的数据；
2. 高频数据采集：秒级以下的高频指标采集，避免周期轮询的性能开销，直接推送数据；
3. 单向网络隔离场景：仅允许被监控端主动访问 Server，Server 无法主动访问被监控端的网络环境；
4. 批量数据上报：自定义脚本批量生成的多维度指标，一次性推送给 Server，提升采集效率；
5. 离线数据补传：网络中断期间缓存的监控数据，网络恢复后批量补传给 Server。

### 3.4.3 zabbix_sender 命令行工具

`zabbix_sender`是 Zabbix 官方提供的 Trapper 模式标准数据推送工具，支持单条数据推送与批量数据推送，是 Trapper 模式的核心运维工具。

#### 3.4.3.1 核心语法与参数

基础命令格式：

```bash
zabbix_sender -z <Server/Proxy IP> -p <端口> -s <主机名> -k <Trapper监控项Key> -o <指标值>
```

核心高频参数：

| 参数  | 全称                | 功能说明                                 |
| --- | ----------------- | ------------------------------------ |
| -z  | --zabbix-server   | Zabbix Server/Proxy 的 IP 地址，必填       |
| -p  | --port            | Server/Proxy 的 Trapper 监听端口，默认 10051 |
| -s  | --host            | Zabbix 中配置的主机名，必须完全一致，必填             |
| -k  | --key             | Trapper 类型监控项的 Key 名称，必填             |
| -o  | --value           | 推送的指标值，必填                            |
| -i  | --input-file      | 批量推送的输入文件路径，支持批量多指标推送                |
| -T  | --with-timestamps | 输入文件中包含时间戳，支持历史数据补传                  |
| -v  | --verbose         | 详细输出模式，用于故障排查                        |

#### 3.4.3.2 批量推送用法

批量推送是生产环境的高频用法，通过输入文件一次性推送多个监控项、多个时间点的数据，输入文件格式为：

```shell
<主机名> <Key名称> <时间戳> <指标值>
```

示例输入文件`data.txt`：

```bash
web-server-01 mysql.connections 1712345600 125
web-server-01 mysql.qps 1712345600 2340
web-server-01 system.cpu.util 1712345600 32.5
```

批量推送命令：

```bash
zabbix_sender -z 192.168.1.10 -s web-server-01 -i data.txt -T
```

### 3.4.4 配置约束与注意事项

1. Zabbix 中必须预先创建对应主机、类型为 “Zabbix Trapper” 的监控项，否则 Server 会拒绝接收推送数据；
2. 推送的主机名必须与 Zabbix 中配置的主机名完全一致，大小写敏感；
3. 推送的指标值必须与监控项定义的数据类型完全匹配，否则数据会被丢弃；
4. 生产环境需配置 Server 的 Trapper 访问控制，限制允许推送数据的 IP 地址，避免恶意数据注入；
5. 批量推送时需控制单次推送的数据量，避免单次推送过大导致 Server 性能波动。
