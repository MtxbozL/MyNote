
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

```ini
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

```ini
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
