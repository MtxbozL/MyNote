# 第十章 高可用集群架构

本章聚焦 Nginx 接入层的高可用解决方案，核心解决 Nginx 单点故障导致的全业务中断风险，覆盖从单点故障根源分析、VRRP 协议核心原理、Keepalived 高可用组件，到企业级主流的**主备模式**与**双主模式**集群部署，再到生产级健康检查与故障自愈体系的完整技术链路。所有内容严格遵循 TCP/IP 协议规范与企业级运维最佳实践，是实现 Nginx 服务 7×24 小时无间断运行的核心技术支撑。

本章核心组件 Keepalived 为 Linux 系统原生开源高可用软件，无需 Nginx 额外编译模块，所有配置均兼容 CentOS/RHEL/Debian/Ubuntu 等主流 Linux 发行版，适配生产环境全场景部署需求。

---

## 01 单点故障问题

### 1.1 单点故障的核心定义与业务影响

在 Web 服务架构中，Nginx 通常作为整个业务体系的**唯一流量入口**，承担反向代理、负载均衡、SSL 卸载、安全防护等核心职责，所有客户端请求必须经过 Nginx 才能转发至后端应用服务、数据库、缓存等资源。

**单点故障（Single Point of Failure, SPOF）**：指架构中唯一的 Nginx 节点，因任何异常导致服务不可用时，整个业务体系的流量入口完全中断，无论后端应用集群的可用性多高，终端用户都无法访问业务服务，造成全业务瘫痪。

### 1.2 单点故障的典型触发场景

Nginx 单点故障的触发场景覆盖硬件、系统、软件、网络全维度，生产环境中无冗余的单点架构无法规避任何一类风险：

1. **硬件故障**：服务器 CPU、内存、磁盘、主板物理损坏，机房电源、制冷系统故障，导致服务器完全宕机；
2. **系统故障**：Linux 操作系统内核崩溃、文件系统损坏、系统资源耗尽（OOM）、网络栈异常，导致服务器无法正常响应；
3. **软件故障**：Nginx 配置错误导致服务无法启动 / 重载、进程崩溃、内存泄漏、模块异常，导致 Nginx 服务停止响应；
4. **网络故障**：机房交换机故障、链路中断、IP 地址冲突、防火墙规则异常，导致 Nginx 节点网络不可达；
5. **运维风险**：人为误操作删除配置、误执行停止命令、变更操作未验证导致服务异常，是生产环境高频故障诱因。

### 1.3 单点架构的核心缺陷

表格

|缺陷维度|具体表现|业务影响|
|---|---|---|
|可用性上限|无冗余容错能力，节点故障即业务中断，可用性最高仅 99.9%，无法满足金融、电商等核心业务 99.99% 以上的可用性要求|故障停机时间长，年度允许停机时间仅 8.76 小时，无法支撑核心业务连续性要求|
|运维能力受限|无法实现无损变更、滚动升级，任何配置变更、版本升级都需要停止服务，存在业务中断窗口|变更风险极高，只能在业务低峰期操作，运维效率低下，无法应对紧急漏洞修复需求|
|性能瓶颈|所有流量集中在单节点，无法实现流量横向扩展，业务增长时只能垂直升级服务器硬件，存在性能天花板|高并发场景下易出现性能瓶颈，无法应对业务峰值流量，易引发雪崩效应|
|容灾能力缺失|无法应对机房级、机柜级故障，单机房断电 / 断网即全业务中断|无地域级容灾能力，不符合等保 2.0、金融行业合规性要求|

### 1.4 高可用架构的核心设计目标

Nginx 高可用集群架构的核心目标，是通过冗余节点设计，彻底消除单点故障，实现：

1. **故障无感知切换**：主节点故障时，备用节点在毫秒级自动接管流量，客户端 TCP 连接无中断，业务访问无感知，RTO（恢复时间目标）趋近于 0；
2. **数据零丢失**：切换过程中无请求丢失、无数据错乱，RPO（恢复点目标）=0；
3. **无损运维能力**：支持配置变更、版本升级、硬件维护的无损操作，无需停止业务服务；
4. **线性扩展能力**：支持集群节点横向扩展，可通过增加节点提升集群承载能力与可用性；
5. **故障自愈能力**：内置健康检查与自动切换机制，无需人工干预即可完成故障转移，大幅降低故障处理时长。

---

## 02 Keepalived 介绍

### 2.1 核心底层：VRRP 虚拟路由冗余协议

Keepalived 的高可用能力完全基于**VRRP（Virtual Router Redundancy Protocol，虚拟路由冗余协议）** 实现，该协议是 IETF 制定的局域网网关冗余标准协议（RFC 3768），核心设计目标是解决局域网内默认网关的单点故障问题，后被广泛应用于 LVS、Nginx 等接入层服务的高可用场景。

#### 2.1.1 VRRP 核心概念

表格

|概念名称|核心定义|
|---|---|
|VRRP 路由器组|由多台运行 VRRP 协议的服务器组成的高可用集群，分为**主节点（Master）** 和**备节点（Backup）**，同一组内节点角色互斥|
|VRID（Virtual Router ID）|虚拟路由器 ID，是 VRRP 组的唯一标识，取值范围 0-255，同一 VRRP 组内所有节点的 VRID 必须完全一致，同一局域网内不同 VRRP 组的 VRID 不可重复，否则会引发协议冲突|
|虚拟 IP（VIP）|虚拟 IP 地址，是 VRRP 组对外提供服务的统一入口地址，同一时间仅由主节点持有，主节点故障时自动漂移至备节点，客户端仅需访问 VIP，无需感知后端真实节点|
|优先级（Priority）|节点的选举优先级，取值范围 1-255，数值越大优先级越高，同一 VRRP 组内主节点优先级必须高于备节点，优先级最高的节点将被选举为主节点|
|VRRP 广播报文|主节点通过组播地址`224.0.0.18`（IP 协议号 112）定时发送 VRRP 广播报文，宣告自身存活状态与优先级，报文发送间隔默认 1 秒|
|抢占模式|主节点故障恢复后，若自身优先级高于当前主节点，会自动抢占回主节点角色，重新接管 VIP；非抢占模式下，故障恢复后仍保持备节点角色，仅当当前主节点故障时才会接管|

#### 2.1.2 VRRP 核心工作机制

1. **角色选举阶段**：VRRP 组启动时，所有节点发送 VRRP 广播报文，对比优先级，优先级最高的节点被选举为主节点，其余节点为备节点；
2. **主节点运行阶段**：主节点绑定 VIP，对外提供服务，定时发送 VRRP 广播报文，向备节点宣告自身存活状态；
3. **备节点监听阶段**：备节点持续监听主节点的 VRRP 广播报文，若在 3 个报文发送周期（默认 3 秒）内未收到主节点的报文，即判定主节点故障；
4. **故障切换阶段**：主节点故障后，备节点重新发起选举，优先级最高的备节点切换为主节点，发送**免费 ARP 报文**，向局域网宣告 VIP 对应的 MAC 地址已变更为当前节点的网卡 MAC，交换机更新 MAC 地址表，流量自动切换至新主节点，整个切换过程耗时 < 1 秒，客户端无感知；
5. **故障恢复阶段**：原主节点故障恢复后，根据抢占模式配置，决定是否重新抢占主节点角色。

### 2.2 Keepalived 组件与核心能力

Keepalived 是基于 VRRP 协议实现的轻量级开源高可用软件，专为 Linux 环境下的 LVS、Nginx 等接入层服务设计，原生集成了 VRRP 协议实现、服务健康检查、故障通知等核心能力，是企业级 Nginx 高可用集群的行业标准方案。

#### 2.2.1 Keepalived 核心组件架构

Keepalived 采用模块化设计，核心组件分为 4 层，各司其职：

表格

|组件层级|组件名称|核心功能|
|---|---|---|
|核心协议层|VRRP Stack|完整实现 VRRP 协议标准，负责节点角色选举、VIP 管理、故障切换、报文收发，是高可用能力的核心基础|
|健康检查层|Checkers|内置多维度健康检查模块，支持 TCP 端口检查、HTTP/HTTPS 服务可用性检查、自定义 Shell 脚本检查，可实时监控服务状态，动态调整节点 VRRP 优先级，触发故障切换|
|系统交互层|System Call & IPVS Wrapper|提供系统调用接口，执行自定义脚本、发送信号、管理 IPVS 规则，实现与 Linux 内核、Nginx 服务的深度交互|
|通知告警层|Notification|内置邮件、脚本通知机制，节点角色切换、故障发生时，可触发自定义脚本发送告警通知至运维人员|

#### 2.2.2 Keepalived 核心优势

1. **轻量稳定**：纯 C 语言开发，资源占用极低，无额外依赖，运行稳定，经过全球企业级生产环境数十年验证；
2. **原生适配 Nginx**：无需修改 Nginx 源码与配置，无需额外模块，可无缝实现 Nginx 集群的高可用；
3. **灵活的健康检查**：支持自定义脚本实现任意维度的健康检查，可适配 Nginx、后端服务、系统资源等全场景监控；
4. **无感知故障切换**：基于二层网络的 VIP 漂移，切换过程毫秒级完成，客户端 TCP 连接无中断，业务完全无感知；
5. **开源免费**：完全开源，无商业 license 限制，社区活跃，文档完善，运维成本极低。

---

## 03 Nginx + Keepalived 双机热备（主备模式）

主备模式（Master-Backup）是企业级生产环境最常用、最稳定的 Nginx 高可用部署方案，核心架构为「一主一备」双节点冗余，主节点承担所有业务流量，备节点实时监控主节点状态，主节点故障时自动接管所有流量，彻底消除单点故障。

### 3.1 架构设计与环境规划

#### 3.1.1 核心架构逻辑

1. 集群由两台物理 / 虚拟服务器组成，分别为**主节点（Master）** 和**备节点（Backup）**；
2. 两台服务器均部署相同版本的 Nginx，配置文件完全一致，保证切换后业务行为无差异；
3. 两台服务器均部署 Keepalived，组成同一个 VRRP 组，对外提供唯一的虚拟 IP（VIP）作为业务访问入口；
4. 正常运行时，主节点持有 VIP，处理所有客户端流量，备节点处于待机状态，不处理业务流量；
5. 主节点故障时，备节点在毫秒级自动接管 VIP，切换为主节点，处理所有业务流量；
6. 主节点故障恢复后，可通过配置选择「抢占模式」自动切回主角色，或「非抢占模式」保持备角色，避免频繁切换引发业务波动。

#### 3.1.2 标准化环境规划

表格

|节点角色|服务器主机名|真实 IP 地址|虚拟 IP（VIP）|VRID|默认优先级|
|---|---|---|---|---|---|
|主节点 Master|nginx-master|192.168.1.10|192.168.1.100|51|150|
|备节点 Backup|nginx-backup|192.168.1.11|192.168.1.100|51|100|

#### 3.1.3 前置环境准备

1. 两台服务器均完成 Nginx 安装部署，配置文件同步一致，服务可正常启动访问；
2. 两台服务器关闭 SELinux（`setenforce 0`，并修改`/etc/selinux/config`永久关闭），避免阻止 VIP 绑定与 VRRP 报文收发；
3. 防火墙放行 VRRP 组播流量，执行以下命令永久生效：
    
    bash
    
    运行
    
    ```
    # firewalld 防火墙
    firewall-cmd --add-rich-rule='rule protocol value="vrrp" accept' --permanent
    firewall-cmd --reload
    # iptables 防火墙
    iptables -A INPUT -p vrrp -j ACCEPT
    service iptables save
    ```
    
4. 两台服务器安装 Keepalived，包管理器直接安装即可：
    
    bash
    
    运行
    
    ```
    # CentOS/RHEL
    yum install -y keepalived
    # Debian/Ubuntu
    apt install -y keepalived
    ```
    

### 3.2 核心配置文件详解

Keepalived 的核心配置文件为`/etc/keepalived/keepalived.conf`，主备节点配置需严格对应，核心差异为节点角色、优先级配置。

#### 3.2.1 主节点（Master）标准配置

ini

```
! 全局配置块
global_defs {
    ! 路由器标识，集群内唯一，建议使用主机名
    router_id nginx-master
    ! 关闭邮件通知，如需启用可配置smtp服务器
    notification_email {
        # ops@example.com
    }
    enable_script_security
    script_user root root
}

! VRRP脚本定义块：Nginx健康检查脚本
vrrp_script check_nginx {
    ! 健康检查脚本路径
    script "/etc/keepalived/check_nginx.sh"
    ! 脚本执行间隔，单位秒
    interval 2
    ! 脚本检查失败时，节点优先级降低的数值
    weight -60
    ! 连续2次检查失败，才判定为异常
    fall 2
    ! 连续2次检查成功，判定为恢复正常
    rise 2
}

! VRRP实例配置块：核心高可用配置
vrrp_instance VI_1 {
    ! 节点初始角色，主节点为MASTER，备节点为BACKUP
    state MASTER
    ! 绑定的物理网卡名称，需与服务器实际网卡一致（可通过ip addr命令查看）
    interface ens33
    ! VRRP组ID，主备节点必须完全一致
    virtual_router_id 51
    ! 节点优先级，主节点必须高于备节点，差值建议≥50
    priority 150
    ! VRRP广播报文发送间隔，单位秒，默认1秒
    advert_int 1
    ! 开启非抢占模式，主节点故障恢复后不自动抢占，需配合state BACKUP使用
    ! nopreempt
    ! 认证配置，主备节点必须完全一致，防止非法节点加入VRRP组
    authentication {
        auth_type PASS
        auth_pass nginx@2026
    }
    ! 虚拟IP配置，可配置多个，每行一个
    virtual_ipaddress {
        192.168.1.100/24 dev ens33 label ens33:0
    }
    ! 引用健康检查脚本，实时调整节点优先级
    track_script {
        check_nginx
    }
    ! 故障切换触发的通知脚本
    ! notify_master "/etc/keepalived/notify.sh master"
    ! notify_backup "/etc/keepalived/notify.sh backup"
    ! notify_fault "/etc/keepalived/notify.sh fault"
}
```

#### 3.2.2 备节点（Backup）标准配置

备节点配置与主节点 90% 一致，仅需修改 3 个核心参数，其余配置完全相同，避免配置差异导致切换异常：

ini

```
! 全局配置块
global_defs {
    router_id nginx-backup
    enable_script_security
    script_user root root
}

! VRRP脚本定义块：与主节点完全一致
vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight -60
    fall 2
    rise 2
}

! VRRP实例配置块
vrrp_instance VI_1 {
    ! 备节点初始角色为BACKUP
    state BACKUP
    interface ens33
    ! VRID与主节点完全一致
    virtual_router_id 51
    ! 优先级低于主节点
    priority 100
    advert_int 1
    ! 认证配置与主节点完全一致
    authentication {
        auth_type PASS
        auth_pass nginx@2026
    }
    ! VIP与主节点完全一致
    virtual_ipaddress {
        192.168.1.100/24 dev ens33 label ens33:0
    }
    ! 引用健康检查脚本，与主节点一致
    track_script {
        check_nginx
    }
}
```

### 3.3 配置关键参数解析

1. **state 角色配置**：`state MASTER/BACKUP`仅为节点初始角色，最终角色由优先级选举决定，即使配置为`BACKUP`，若优先级高于其他节点，仍会被选举为主节点；
2. **interface 网卡配置**：必须填写服务器实际的物理网卡名称，可通过`ip addr`命令查看，常见名称为`ens33`、`eth0`、`ens192`，配置错误会导致 VIP 无法绑定；
3. **virtual_router_id VRID 配置**：同一 VRRP 组主备节点必须完全一致，同一局域网内不同 VRRP 组不可重复，否则会引发 VRRP 协议冲突，导致脑裂；
4. **priority 优先级配置**：主节点优先级必须高于备节点，差值建议≥50，保证健康检查失败时，主节点优先级降低后低于备节点，触发切换；
5. **weight 权重配置**：健康检查失败时，优先级降低的数值，必须保证主节点降低后的优先级低于备节点，例如主节点优先级 150，weight -60，降低后为 90，低于备节点的 100，可正常触发切换；
6. **nopreempt 非抢占模式**：生产环境推荐开启非抢占模式，避免主节点故障恢复后频繁切换，导致业务波动；开启后，两个节点的`state`均需配置为`BACKUP`，仅通过优先级区分角色。

### 3.4 服务启动与验证

1. **配置健康检查脚本**：主备节点均需部署`/etc/keepalived/check_nginx.sh`脚本，脚本内容详见本章 05 节，添加执行权限：
    
    bash
    
    运行
    
    ```
    chmod +x /etc/keepalived/check_nginx.sh
    ```
    
2. **启动 Keepalived 服务**：主备节点均启动服务，并设置开机自启：
    
    bash
    
    运行
    
    ```
    systemctl start keepalived
    systemctl enable keepalived
    ```
    
3. **状态验证**：
    
    - 查看服务运行状态：`systemctl status keepalived`
    - 查看 VIP 绑定情况：主节点执行`ip addr`，可看到 VIP 已绑定到对应网卡
    - 查看系统日志：`tail -f /var/log/messages`，可看到 VRRP 角色选举过程
    
4. **业务访问验证**：客户端访问 VIP，可正常访问 Nginx 服务，与直接访问主节点真实 IP 行为一致。

### 3.5 故障切换测试

生产环境上线前，必须完成全场景故障切换测试，验证高可用能力：

1. **主节点 Keepalived 停止测试**：主节点执行`systemctl stop keepalived`，查看 VIP 是否漂移至备节点，业务访问是否正常；
2. **主节点 Nginx 进程异常测试**：主节点执行`nginx -s stop`，验证健康检查脚本是否触发优先级降低，VIP 是否漂移至备节点；
3. **主节点服务器宕机测试**：强制关闭主节点服务器，验证 VIP 是否自动漂移，业务是否无中断；
4. **故障恢复测试**：主节点恢复正常后，验证抢占 / 非抢占模式是否符合预期，业务无异常波动。

---

## 04 Nginx + Keepalived 双主模式（互为主备）

双主模式是主备模式的进阶优化方案，核心解决主备模式下备节点资源长期闲置的问题，通过两个独立的 VRRP 实例，实现两台服务器**互为主备**，正常运行时两台节点同时处理业务流量，分摊负载；单节点故障时，另一节点接管所有流量，既消除了单点故障，又充分利用了服务器硬件资源，是高流量业务场景的首选方案。

### 4.1 架构设计与环境规划

#### 4.1.1 核心架构逻辑

1. 集群由两台服务器组成，创建**两个独立的 VRRP 实例**，每个实例对应一个 VIP；
2. **VRRP 实例 1**：节点 1 为 Master，节点 2 为 Backup，对应 VIP1，负责业务 A 流量；
3. **VRRP 实例 2**：节点 2 为 Master，节点 1 为 Backup，对应 VIP2，负责业务 B 流量；
4. 正常运行时，两台服务器分别持有各自 Master 实例的 VIP，同时处理业务流量，实现负载分摊；
5. 任意一台服务器故障时，另一台服务器自动接管两个实例的 VIP，处理所有业务流量，保证业务不中断；
6. 故障节点恢复后，自动切回原角色，两台服务器重新分摊流量。

#### 4.1.2 标准化环境规划

表格

|服务器真实 IP|VRRP 实例 1 角色|VRRP 实例 1 VIP|VRRP 实例 1 VRID|实例 1 优先级|VRRP 实例 2 角色|VRRP 实例 2 VIP|VRRP 实例 2 VRID|实例 2 优先级|
|---|---|---|---|---|---|---|---|---|
|192.168.1.10|MASTER|192.168.1.100|51|150|BACKUP|192.168.1.101|52|100|
|192.168.1.11|BACKUP|192.168.1.100|51|100|MASTER|192.168.1.101|52|150|

#### 4.1.3 流量分流实现

双主模式的流量分流通常通过两种方式实现：

1. **DNS 轮询解析**：将业务域名同时解析到 VIP1 和 VIP2，DNS 服务器采用轮询策略返回不同的 VIP 地址，客户端随机访问其中一个 VIP，实现流量自动分流；
2. **多域名分流**：不同业务域名解析到不同的 VIP，例如`www.example.com`解析到 VIP1，`api.example.com`解析到 VIP2，实现业务维度的流量隔离与分摊。

### 4.2 核心配置文件详解

双主模式的核心是在同一配置文件中定义两个独立的 VRRP 实例，两个实例的 VRID 必须不同，避免协议冲突，节点角色与优先级互为镜像。

#### 4.2.1 节点 1（192.168.1.10）标准配置

ini

```
global_defs {
    router_id nginx-node1
    enable_script_security
    script_user root root
}

! 健康检查脚本：两个实例共用同一个脚本
vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight -60
    fall 2
    rise 2
}

! VRRP实例1：节点1为MASTER
vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass nginx_vip1@2026
    }
    virtual_ipaddress {
        192.168.1.100/24 dev ens33 label ens33:0
    }
    track_script {
        check_nginx
    }
}

! VRRP实例2：节点1为BACKUP
vrrp_instance VI_2 {
    state BACKUP
    interface ens33
    virtual_router_id 52
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass nginx_vip2@2026
    }
    virtual_ipaddress {
        192.168.1.101/24 dev ens33 label ens33:1
    }
    track_script {
        check_nginx
    }
}
```

#### 4.2.2 节点 2（192.168.1.11）标准配置

ini

```
global_defs {
    router_id nginx-node2
    enable_script_security
    script_user root root
}

! 健康检查脚本：与节点1完全一致
vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight -60
    fall 2
    rise 2
}

! VRRP实例1：节点2为BACKUP
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass nginx_vip1@2026
    }
    virtual_ipaddress {
        192.168.1.100/24 dev ens33 label ens33:0
    }
    track_script {
        check_nginx
    }
}

! VRRP实例2：节点2为MASTER
vrrp_instance VI_2 {
    state MASTER
    interface ens33
    virtual_router_id 52
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass nginx_vip2@2026
    }
    virtual_ipaddress {
        192.168.1.101/24 dev ens33 label ens33:1
    }
    track_script {
        check_nginx
    }
}
```

### 4.3 核心注意事项

1. **VRID 必须唯一**：两个 VRRP 实例的`virtual_router_id`必须不同，且不能与局域网内其他 VRRP 组的 VRID 重复，否则会引发协议冲突，导致脑裂与 VIP 冲突；
2. **认证密码独立**：两个 VRRP 实例建议使用不同的认证密码，避免相互干扰；
3. **网卡别名区分**：两个 VIP 需配置不同的网卡别名（如`ens33:0`、`ens33:1`），便于区分与管理；
4. **Nginx 配置一致性**：两台服务器的 Nginx 配置必须完全一致，保证两个 VIP 访问的业务行为完全相同，故障切换时无业务差异；
5. **资源预留**：两台服务器的硬件配置需保持一致，且单节点需预留足够的资源余量，保证单节点故障时，另一台服务器可承载全量业务流量，不会出现性能瓶颈。

---

## 05 高可用健康检查脚本

Keepalived 原生的 VRRP 机制仅能监控自身进程与服务器网络状态，**无法感知 Nginx 服务的运行状态**—— 若 Nginx 进程崩溃、服务异常无法响应，但 Keepalived 进程仍正常运行，主节点会继续持有 VIP，导致客户端流量进入后无法处理，业务中断。

自定义健康检查脚本是解决该问题的核心方案，也是企业级生产环境必须配置的核心能力，可实时监控 Nginx 服务状态，异常时自动调整节点优先级，触发 VIP 漂移，实现故障自愈。

### 5.1 健康检查的核心设计目标

1. **全维度监控**：不仅监控 Nginx 进程是否存活，还要监控 Nginx 服务是否可正常响应请求，避免进程存活但服务僵死的情况；
2. **故障自愈**：Nginx 异常时，先尝试自动重启服务，无需人工干预，减少故障切换次数；
3. **自动切换**：重启失败后，自动降低当前节点的 VRRP 优先级，触发 VIP 漂移至备节点，保证业务不中断；
4. **状态恢复**：Nginx 服务恢复正常后，自动恢复节点优先级，保证集群角色符合预期；
5. **日志记录**：完整记录检查过程、异常事件、操作结果，便于故障溯源与审计。

### 5.2 标准 Nginx 健康检查脚本

脚本路径：`/etc/keepalived/check_nginx.sh`，主备节点需完全一致，添加执行权限`chmod +x /etc/keepalived/check_nginx.sh`。

bash

运行

```
#!/bin/bash
# Nginx高可用健康检查脚本
# 日志文件路径
LOG_FILE="/var/log/keepalived_check_nginx.log"
# Nginx PID文件路径，需与nginx.conf中的pid指令一致
NGINX_PID="/usr/local/nginx/logs/nginx.pid"
# 健康检查地址，使用本地回环地址，避免网络波动影响
CHECK_URL="http://127.0.0.1/nginx_status"
# 最大重启尝试次数
MAX_RETRY=2
# 当前重试次数
RETRY_COUNT=0

# 日志输出函数
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1" >> $LOG_FILE
}

# 步骤1：检查Nginx进程是否存活
check_process() {
    if [ -f $NGINX_PID ]; then
        NGINX_PID_VALUE=$(cat $NGINX_PID)
        if kill -0 $NGINX_PID_VALUE > /dev/null 2>&1; then
            log "Nginx进程正常，PID: $NGINX_PID_VALUE"
            return 0
        else
            log "Nginx PID文件存在，但进程已不存在"
            return 1
        fi
    else
        log "Nginx PID文件不存在，进程未启动"
        return 1
    fi
}

# 步骤2：检查Nginx服务是否可正常响应
check_service() {
    HTTP_STATUS=$(curl -o /dev/null -s -w "%{http_code}" --connect-timeout 2 --max-time 3 $CHECK_URL)
    if [ $HTTP_STATUS -eq 200 ]; then
        log "Nginx服务响应正常，HTTP状态码: $HTTP_STATUS"
        return 0
    else
        log "Nginx服务响应异常，HTTP状态码: $HTTP_STATUS"
        return 1
    fi
}

# 步骤3：重启Nginx服务
restart_nginx() {
    log "尝试重启Nginx服务，第$((RETRY_COUNT+1))次尝试"
    # 先尝试平滑重载，失败再强制重启
    /usr/local/nginx/sbin/nginx -t > /dev/null 2>&1
    if [ $? -eq 0 ]; then
        /usr/local/nginx/sbin/nginx -s reload
    else
        /usr/local/nginx/sbin/nginx -s stop > /dev/null 2>&1
        sleep 1
        /usr/local/nginx/sbin/nginx
    fi
    sleep 2
    # 检查重启结果
    if check_process && check_service; then
        log "Nginx服务重启成功"
        return 0
    else
        log "Nginx服务重启失败"
        return 1
    fi
}

# 主执行逻辑
log "==================== 开始健康检查 ===================="
# 先检查进程，再检查服务响应
if check_process && check_service; then
    log "Nginx服务完全正常，健康检查通过"
    exit 0
else
    log "Nginx服务异常，进入故障处理流程"
    # 循环尝试重启
    while [ $RETRY_COUNT -lt $MAX_RETRY ]; do
        RETRY_COUNT=$((RETRY_COUNT+1))
        if restart_nginx; then
            exit 0
        fi
    done
    # 重启失败，退出码1，触发Keepalived优先级降低
    log "Nginx重启失败，已达到最大重试次数，触发VIP漂移"
    exit 1
fi
```

### 5.3 脚本核心配置说明

1. **NGINX_PID**：必须与 Nginx 配置文件中`pid`指令指定的路径完全一致，否则进程检查会失效；
2. **CHECK_URL**：建议使用 Nginx 的`stub_status`状态页面，该页面轻量无业务依赖，可真实反映 Nginx 服务的响应能力，需提前在 Nginx 中配置本地可访问的状态页面；
3. **MAX_RETRY**：最大重启尝试次数，建议设置为 2-3 次，避免频繁重启导致服务异常；
4. **超时设置**：curl 命令设置了 2 秒连接超时、3 秒总超时，避免检查脚本阻塞，影响 Keepalived 的 VRRP 报文收发；
5. **退出码规则**：脚本正常退出码为 0，Keepalived 不调整优先级；退出码为 1 时，Keepalived 按照`weight`配置降低节点优先级，触发故障切换。

### 5.4 进阶健康检查能力

生产环境可根据业务需求，扩展健康检查脚本的能力，实现更全面的监控：

1. **后端服务健康检查**：检查 Nginx 反向代理的后端服务可用性，若所有后端节点均宕机，触发 VIP 漂移，避免流量进入后全部返回 502 错误；
2. **系统资源检查**：监控服务器 CPU、内存、磁盘使用率，若资源耗尽导致服务无法正常运行，触发切换；
3. **端口存活检查**：检查 Nginx 监听的 80/443 端口是否正常监听，避免端口未绑定导致的服务异常；
4. **脑裂检测**：脚本中添加脑裂检测逻辑，若发现两个节点同时持有 VIP，自动停止低优先级节点的 Keepalived 服务，避免 IP 冲突。

### 5.5 故障通知脚本

节点角色发生切换时，可通过自定义通知脚本发送告警至运维人员，及时感知故障事件，脚本路径`/etc/keepalived/notify.sh`，添加执行权限：

bash

运行

```
#!/bin/bash
# Keepalived角色切换通知脚本
LOG_FILE="/var/log/keepalived_notify.log"
NOTIFY_TYPE=$1
SERVER_IP=$(hostname -I | awk '{print $1}')
DATETIME=$(date +'%Y-%m-%d %H:%M:%S')

# 日志记录
echo "[$DATETIME] 节点$SERVER_IP角色切换为: $NOTIFY_TYPE" >> $LOG_FILE

# 钉钉/企业微信/邮件告警
# 此处替换为实际的告警接口地址
WEBHOOK_URL="https://oapi.dingtalk.com/robot/send?access_token=xxx"
curl -H "Content-Type: application/json" -X POST $WEBHOOK_URL -d '{
    "msgtype": "text",
    "text": {
        "content": "【Nginx高可用告警】\n时间：'$DATETIME'\n节点IP：'$SERVER_IP'\n事件：节点角色切换为'$NOTIFY_TYPE'\n请及时排查故障！"
    }
}'
```

在 Keepalived 配置文件的`vrrp_instance`块中引用该脚本：

ini

```
notify_master "/etc/keepalived/notify.sh master"
notify_backup "/etc/keepalived/notify.sh backup"
notify_fault "/etc/keepalived/notify.sh fault"
```

---

## 核心风险点：脑裂问题与解决方案

### 脑裂问题定义

脑裂（Split-Brain）是高可用集群最严重的故障，指主备节点之间的心跳链路中断，双方都判定对方故障，同时切换为主节点，持有同一个 VIP，导致局域网内出现 IP 地址冲突，客户端流量被随机分发到两个节点，出现请求错乱、业务中断、数据不一致等严重问题。

### 脑裂问题的核心诱因

1. 防火墙拦截了 VRRP 组播报文，备节点无法收到主节点的 VRRP 广播；
2. 主节点服务器负载过高、CPU 使用率 100%，无法正常发送 VRRP 广播报文；
3. 主备节点之间的物理链路故障，二层网络不通；
4. VRRP 实例的 VRID、认证密码配置不一致，导致报文无法正常解析；
5. 网卡故障、驱动异常，导致 VRRP 报文无法正常收发。

### 脑裂问题的解决方案

1. **严格配置防火墙规则**：明确放行 VRRP 协议流量，避免规则拦截导致的心跳中断；
2. **双链路心跳检测**：除了 VRRP 组播心跳，额外添加 TCP 直连心跳检测，双重校验节点存活状态；
3. **脑裂检测与自动处理**：在健康检查脚本中添加脑裂检测逻辑，通过 arping 命令检测 VIP 是否在其他节点存在，若发现脑裂，自动停止自身 Keepalived 服务，释放 VIP，保证集群角色唯一；
4. **硬件与网络冗余**：主备节点部署在同一机柜的不同交换机下，避免单交换机故障导致的链路中断；
5. **实时告警监控**：配置 VIP 绑定状态监控、节点角色监控，发现脑裂事件立即触发最高级别告警，运维人员第一时间介入处理。