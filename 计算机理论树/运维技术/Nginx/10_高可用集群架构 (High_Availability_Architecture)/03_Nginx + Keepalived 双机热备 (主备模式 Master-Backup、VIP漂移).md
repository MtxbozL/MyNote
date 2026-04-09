
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

|节点角色|服务器主机名|真实 IP 地址|虚拟 IP（VIP）|VRID|默认优先级|
|---|---|---|---|---|---|
|主节点 Master|nginx-master|192.168.1.10|192.168.1.100|51|150|
|备节点 Backup|nginx-backup|192.168.1.11|192.168.1.100|51|100|

#### 3.1.3 前置环境准备

1. 两台服务器均完成 Nginx 安装部署，配置文件同步一致，服务可正常启动访问；
2. 两台服务器关闭 SELinux（`setenforce 0`，并修改`/etc/selinux/config`永久关闭），避免阻止 VIP 绑定与 VRRP 报文收发；
3. 防火墙放行 VRRP 组播流量，执行以下命令永久生效：

    ```bash
    # firewalld 防火墙
    firewall-cmd --add-rich-rule='rule protocol value="vrrp" accept' --permanent
    firewall-cmd --reload
    # iptables 防火墙
    iptables -A INPUT -p vrrp -j ACCEPT
    service iptables save
    ```
    
4. 两台服务器安装 Keepalived，包管理器直接安装即可：
    
    ```bash
    # CentOS/RHEL
    yum install -y keepalived
    # Debian/Ubuntu
    apt install -y keepalived
    ```
    

### 3.2 核心配置文件详解

Keepalived 的核心配置文件为`/etc/keepalived/keepalived.conf`，主备节点配置需严格对应，核心差异为节点角色、优先级配置。

#### 3.2.1 主节点（Master）标准配置

```ini
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

```ini
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

    ```bash
    chmod +x /etc/keepalived/check_nginx.sh
    ```
    
2. **启动 Keepalived 服务**：主备节点均启动服务，并设置开机自启：
    
    ```bash
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
