#### 核心标准答案

##### 一、NAT 的核心定义

NAT 全称**Network Address Translation（网络地址转换）**，是工作在 OSI 模型网络层的技术，核心作用是**修改 IP 数据包的源 IP 地址或目的 IP 地址，实现私网地址与公网地址的转换，解决公网 IPv4 地址枯竭的问题，同时隐藏内网拓扑，提升网络安全性**。

NAT 主要分为三种核心类型：

1. **SNAT（Source NAT，源地址转换）**：修改数据包的源 IP 地址，核心场景是**内网多个设备共享一个公网 IP 访问互联网**，将内网私网 IP 转换为网关的公网 IP，实现内网设备上网。
2. **DNAT（Destination NAT，目的地址转换）**：修改数据包的目的 IP 地址，核心场景是**将公网 IP 的端口映射到内网服务器的端口，让互联网用户可以访问内网的服务**。
3. **PAT（端口地址转换）**：是 NAT 的一种特殊形式，通过端口号的映射，实现多个私网 IP 共用一个公网 IP，是目前最常用的 NAT 方式，SNAT/DNAT 通常都会配合 PAT 使用。

##### 二、Linux 配置 SNAT 和 DNAT

Linux 通过**iptables/netfilter**框架实现 NAT 配置，核心操作 nat 表，具体配置命令如下：

###### 1. 前置准备

- 开启 Linux 内核的 IP 转发功能：
    
    bash
    
    运行
    
    ```
    # 临时开启
    echo 1 > /proc/sys/net/ipv4/ip_forward
    # 永久开启，修改/etc/sysctl.conf，添加
    net.ipv4.ip_forward = 1
    # 生效配置
    sysctl -p
    ```
    
- 确认 iptables 已安装，nat 表的链正常。

###### 2. SNAT 配置

分为两种场景：

- **场景 1：网关有固定公网 IP**
    
    内网网段为 192.168.1.0/24，网关公网 IP 为 114.114.114.114，配置命令：
    
    bash
    
    运行
    
    ```
    # 配置SNAT，将内网网段的源IP转换为公网IP
    iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j SNAT --to-source 114.114.114.114
    ```
    
    说明：`-t nat`指定 nat 表，`POSTROUTING`是数据包发出前的链，`-o eth0`是公网出口网卡，`--to-source`指定转换后的源 IP。
- **场景 2：网关公网 IP 不固定（拨号上网、动态 IP）**
    
    使用 MASQUERADE（地址伪装）模式，自动获取出口网卡的公网 IP，无需手动指定：
    
    bash
    
    运行
    
    ```
    iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE
    ```
    

###### 3. DNAT 配置

场景：公网用户访问网关的 80 端口，转发到内网服务器 192.168.1.100 的 8080 端口，网关公网 IP 为 114.114.114.114，配置命令：

bash

运行

```
# 配置DNAT，修改数据包的目的IP和端口
iptables -t nat -A PREROUTING -d 114.114.114.114 -i eth0 -p tcp --dport 80 -j DNAT --to-destination 192.168.1.100:8080
```

说明：`PREROUTING`是数据包进入网卡后的链，`-i eth0`是公网入口网卡，`--dport`是公网访问的端口，`--to-destination`指定内网服务器的 IP 和端口。

#### 专家级拓展

- 核心规则补充：
    
    1. SNAT 配置后，必须保证内网服务器的网关指向 Linux 网关，否则回包无法正常转发；
    2. DNAT 配置后，需要在 filter 表的 FORWARD 链放通对应的流量，否则数据包会被防火墙拦截：
    
    bash
    
    运行
    
    ```
    # 放通内网服务器的8080端口转发
    iptables -A FORWARD -d 192.168.1.100 -p tcp --dport 8080 -j ACCEPT
    ```
    
    3. 端口映射支持一对一映射，也支持端口范围映射，同时支持 TCP 和 UDP 协议。
    
- 生产环境最佳实践：
    
    1. 配置 NAT 规则后，通过`iptables-save`保存规则，避免服务器重启后规则丢失；
    2. 配置合理的防火墙规则，只放通必要的端口和 IP，避免内网暴露到公网；
    3. 开启 NAT 的连接跟踪功能，保证请求和响应的报文能正确转换，避免连接异常；
    4. 高可用场景下，配合 keepalived 实现 NAT 网关的主备高可用，避免单点故障。
    

#### 面试避坑指南

- 严禁混淆 SNAT 和 DNAT 的作用、使用的链，SNAT 用 POSTROUTING 链，DNAT 用 PREROUTING 链，不能说反，这是面试高频错误；
- 避免遗漏开启 IP 转发，这是 Linux 实现 NAT 的前提，不提会被判定为无实际配置经验；
- 不要只说配置命令，不说 NAT 的核心作用和适用场景，面试问这个问题，核心是考察你对 NAT 的理解，而不只是背命令。