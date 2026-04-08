本文档覆盖**临时 / 永久修改**全场景，适配两大发行版主流网络管理方案，包含生效验证、踩坑避坑、故障排查全流程，可直接作为运维手册使用。

## 前置核心说明（必看，90% 不生效问题源于此）

1. `/etc/resolv.conf` 是 Linux 传统 DNS 配置入口，但现代发行版（Fedora/Debian 10+）默认将其设为软链接，由 `NetworkManager`/`systemd-resolved` 接管，**直接手动修改会在重启网络 / 系统后被覆盖**，仅适用于临时场景。
2. 禁止混用多种网络管理工具，先确认当前系统使用的管理方案，仅用对应方法修改：

| 发行版                    | 主流场景       | 首选管理工具                              |
| ---------------------- | ---------- | ----------------------------------- |
| Fedora 39+/Workstation | 桌面 / 服务器默认 | NetworkManager + systemd-resolved   |
| Fedora 最小化服务器          | 无图形环境      | systemd-networkd + systemd-resolved |
| Debian 11+/ 桌面版        | 桌面默认       | NetworkManager                      |
| Debian 服务器 / 老版本       | 经典运维场景     | ifupdown（/etc/network/interfaces）   |
| Debian 云服务器镜像          | 云环境        | netplan                             |

3. 通用 DNS 查看 & 验证命令（修改前后必用）

    ```bash
    # 查看当前resolv.conf配置
    cat /etc/resolv.conf
    # 查看systemd-resolved接管的DNS详情（推荐）
    resolvectl status
    # 查看NetworkManager接管的DNS
    nmcli device show | grep DNS
    # 测试DNS解析是否生效，确认使用的DNS服务器
    nslookup baidu.com
    dig baidu.com
    ```
    

---

## 一、Fedora 修改 DNS 完整指南

适配 Fedora 39/40/41+ 全版本，按推荐优先级排序。

### 1. 临时修改 DNS（重启网络 / 系统失效）

仅适用于临时测试场景，立即生效，重启后自动还原。

bash

运行

```
# 备份原文件（可选）
sudo cp /etc/resolv.conf /etc/resolv.conf.bak
# 编辑配置文件
sudo vi /etc/resolv.conf
```

写入以下内容（多个 DNS 按优先级从上到下排列）：

conf

```
nameserver 223.5.5.5
nameserver 8.8.8.8
nameserver 114.114.114.114
```

保存退出后立即生效，无需重启服务。

### 2. 永久修改 DNS（推荐方案）

#### 方法 1：NetworkManager nmcli 命令行修改（通用首选）

适配 Fedora 桌面 / 服务器默认环境，图形 / 无图形环境通用，稳定性最高。

1. 查看当前网络连接名称（记住 NAME 列的名称，含空格需加引号）
    
    bash
    
    运行
    
    ```
    nmcli connection show
    ```
    
2. 修改对应连接的 DNS 配置（以`Wired connection 1`有线连接为例）
    
    bash
    
    运行
    
    ```
    # 配置IPv4 DNS，多个地址用英文逗号分隔
    sudo nmcli connection modify "Wired connection 1" ipv4.dns "223.5.5.5,8.8.8.8"
    # 关闭DHCP自动获取DNS，防止自定义配置被覆盖
    sudo nmcli connection modify "Wired connection 1" ipv4.ignore-auto-dns yes
    
    # 如需配置IPv6 DNS，执行以下命令
    sudo nmcli connection modify "Wired connection 1" ipv6.dns "2400:3200::1,2001:4860:4860::8888"
    sudo nmcli connection modify "Wired connection 1" ipv6.ignore-auto-dns yes
    ```
    
3. 重启连接使配置生效
    
    bash
    
    运行
    
    ```
    sudo nmcli connection up "Wired connection 1"
    ```
    
4. 验证生效：`resolvectl status` 查看对应网卡的 DNS 服务器。

#### 方法 2：图形界面修改（Fedora Workstation 桌面版）

适合新手，可视化操作无命令门槛：

1. 打开 `设置 -> 网络`
2. 找到目标网络（有线 / Wi-Fi），点击右侧齿轮图标进入设置
3. 切换到 `IPv4` 选项卡，关闭 `DNS` 右侧的「自动」开关
4. 在 DNS 输入框中填写服务器地址，多个用英文逗号分隔
5. 点击「应用」，重启对应网络连接（开关一次）即可生效

#### 方法 3：systemd-resolved 全局 DNS 配置（纯 systemd 服务器场景）

适合禁用 NetworkManager、纯 systemd 环境的服务器，全局生效。

1. 确认服务状态（Fedora 默认启用）
    
    bash
    
    运行
    
    ```
    systemctl enable --now systemd-resolved
    ```
    
2. 编辑核心配置文件
    
    bash
    
    运行
    
    ```
    sudo vi /etc/systemd/resolved.conf
    ```
    
    找到`[Resolve]`段，取消注释并修改配置：
    
    conf
    
    ```
    [Resolve]
    # 主DNS服务器，多个用空格分隔
    DNS=223.5.5.5 8.8.8.8
    # 备用DNS服务器，主DNS不可用时自动切换
    FallbackDNS=114.114.114.114 1.1.1.1
    # 可选：搜索域
    Domains=local
    # 可选：开启DNSSEC校验
    DNSSEC=allow-downgrade
    # 可选：开启DNS over TLS加密
    DNSOverTLS=opportunistic
    ```
    
3. 重启服务使配置生效
    
    bash
    
    运行
    
    ```
    sudo systemctl restart systemd-resolved
    ```
    
4. 修复 /etc/resolv.conf 软链接（核心步骤，否则不生效）
    
    bash
    
    运行
    
    ```
    # 备份原文件
    sudo mv /etc/resolv.conf /etc/resolv.conf.bak
    # 创建软链接（stub模式为本地127.0.0.53转发，推荐）
    sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
    # 如需直接写入DNS到resolv.conf，用upstream模式
    # sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
    ```
    
5. 验证生效：`resolvectl status` 查看 Global 段的 DNS 配置。

#### 方法 4：systemd-networkd 网卡级 DNS 配置（最小化服务器场景）

适配禁用 NetworkManager、使用 systemd-networkd 管理网络的极简环境。

1. 启用 systemd-networkd 服务
    
    bash
    
    运行
    
    ```
    sudo systemctl enable --now systemd-networkd
    ```
    
2. 查看网卡名称（`ip addr`），创建 / 编辑网卡配置文件
    
    bash
    
    运行
    
    ```
    sudo vi /etc/systemd/network/20-wired.network
    ```
    
3. 写入对应配置（二选一）
    
    - DHCP 自动获取 IP 场景
        
        ini
        
        ```
        [Match]
        Name=ens33  # 替换为你的网卡名称
        
        [Network]
        DHCP=yes
        DNS=223.5.5.5
        DNS=8.8.8.8
        
        [DHCPv4]
        UseDNS=no  # 禁用DHCP自动获取DNS
        
        [DHCPv6]
        UseDNS=no
        ```
        
    - 静态 IP 场景
        
        ini
        
        ```
        [Match]
        Name=ens33  # 替换为你的网卡名称
        
        [Network]
        Address=192.168.1.100/24
        Gateway=192.168.1.1
        DNS=223.5.5.5
        DNS=8.8.8.8
        ```
        
    
4. 重启服务生效
    
    bash
    
    运行
    
    ```
    sudo systemctl restart systemd-networkd
    ```
    
5. 配合 systemd-resolved 配置软链接（同方法 3 步骤 4），验证生效。

---

## 二、Debian 修改 DNS 完整指南

适配 Debian 10 (Buster)/11 (Bullseye)/12 (Bookworm) 全版本，覆盖经典 / 现代运维场景。

### 1. 临时修改 DNS（重启网络 / 系统失效）

与 Fedora 完全一致，直接编辑`/etc/resolv.conf`，写入 nameserver 配置，保存即生效，重启后还原。

### 2. 永久修改 DNS（推荐方案）

#### 方法 1：NetworkManager nmcli 命令行修改（Debian 桌面版首选）

Debian 桌面版默认预装 NetworkManager，操作逻辑与 Fedora 完全一致，无兼容差异。

1. 查看连接名：`nmcli connection show`
2. 修改 DNS 配置（替换为你的连接名）
    
    bash
    
    运行
    
    ```
    sudo nmcli connection modify "你的连接名称" ipv4.dns "223.5.5.5,8.8.8.8"
    sudo nmcli connection modify "你的连接名称" ipv4.ignore-auto-dns yes
    # IPv6配置同Fedora
    ```
    
3. 重启连接生效：`sudo nmcli connection up "你的连接名称"`
4. 图形界面修改逻辑与 Fedora 完全一致，可直接参考。

#### 方法 2：ifupdown 经典方案（/etc/network/interfaces，服务器首选）

Debian 全版本兼容，是服务器运维最经典、最稳定的方案，老版本默认使用。

1. 查看网卡名称：`ip addr`（如 eth0、ens33）
2. 安装依赖包（核心，否则配置不生效）
    
    bash
    
    运行
    
    ```
    sudo apt update && sudo apt install -y resolvconf
    ```
    
3. 编辑网络配置文件
    
    bash
    
    运行
    
    ```
    sudo vi /etc/network/interfaces
    ```
    
4. 写入对应配置（二选一，仅修改目标网卡，不要改动其他配置）
    
    - DHCP 自动获取 IP 场景
        
        conf
        
        ```
        auto eth0  # 替换为你的网卡名称
        iface eth0 inet dhcp
            dns-nameservers 223.5.5.5 8.8.8.8 114.114.114.114
        # 多个DNS用空格分隔，行首必须缩进
        ```
        
    - 静态 IP 场景
        
        conf
        
        ```
        auto eth0  # 替换为你的网卡名称
        iface eth0 inet static
            address 192.168.1.100/24
            gateway 192.168.1.1
            netmask 255.255.255.0
            dns-nameservers 223.5.5.5 8.8.8.8
        ```
        
    
5. 重启网络生效（推荐单网卡重启，避免远程断连）
    
    bash
    
    运行
    
    ```
    # 单网卡重启（推荐，不影响其他网卡）
    sudo ifdown eth0 && sudo ifup eth0
    # 全网络服务重启
    sudo systemctl restart networking
    ```
    
6. 验证生效：`cat /etc/resolv.conf` 会自动生成对应的 nameserver 配置。

#### 方法 3：systemd-resolved 全局配置（Debian 11+ 推荐）

适配 Debian 10+ 版本，操作逻辑与 Fedora 完全一致，全局生效，兼容所有网络管理工具。

1. 安装并启用服务
    
    bash
    
    运行
    
    ```
    sudo apt update && sudo apt install -y systemd-resolved
    sudo systemctl enable --now systemd-resolved
    ```
    
2. 编辑`/etc/systemd/resolved.conf`配置文件，修改 DNS 参数（同 Fedora 方法 3）
3. 重启服务：`sudo systemctl restart systemd-resolved`
4. 配置`/etc/resolv.conf`软链接（同 Fedora 方法 3 步骤 4，核心必做）
5. 验证生效：`resolvectl status`

#### 方法 4：netplan 方案（云服务器 Debian 镜像首选）

适配 Debian 11+ 版本，主流云服务器 Debian 镜像默认使用，配置简洁，兼容性强。

1. 安装 netplan（最小化镜像可能未预装）
    
    bash
    
    运行
    
    ```
    sudo apt update && sudo apt install -y netplan.io
    ```
    
2. 查看配置文件（文件名通常为 00-installer.yaml/01-netcfg.yaml）
    
    bash
    
    运行
    
    ```
    ls /etc/netplan/
    ```
    
3. 编辑配置文件（**yaml 严格缩进，必须用空格，禁止用 tab，否则直接报错**）
    
    bash
    
    运行
    
    ```
    sudo vi /etc/netplan/01-netcfg.yaml
    ```
    
4. 写入对应配置（二选一）
    
    - DHCP 自动获取 IP 场景
        
        yaml
        
        ```
        network:
          version: 2
          renderer: networkd  # 桌面版可改为NetworkManager
          ethernets:
            ens33:  # 替换为你的网卡名称
              dhcp4: yes
              nameservers:
                addresses: [223.5.5.5, 8.8.8.8, 114.114.114.114]
              dhcp4-overrides:
                use-dns: no  # 禁用DHCP自动获取DNS，防止覆盖
        ```
        
    - 静态 IP 场景
        
        yaml
        
        ```
        network:
          version: 2
          renderer: networkd
          ethernets:
            ens33:  # 替换为你的网卡名称
              addresses:
                - 192.168.1.100/24
              routes:
                - to: default
                  via: 192.168.1.1
              nameservers:
                addresses: [223.5.5.5, 8.8.8.8]
        ```
        
    
5. 测试配置（**远程服务器必做，配置错误会自动回滚，避免断连**）
    
    bash
    
    运行
    
    ```
    sudo netplan try
    ```
    
    测试无报错后，按回车确认生效。
6. 正式应用配置
    
    bash
    
    运行
    
    ```
    sudo netplan apply
    ```
    
7. 验证生效：`resolvectl status` 或 `cat /etc/resolv.conf`

---

## 三、通用排障指南（修改不生效 / 解析失败必看）

1. **配置冲突排查**
    
    禁止同时启用 NetworkManager、networking、systemd-networkd 多个服务，仅保留一个网络管理服务，关闭其他冗余服务：
    
    bash
    
    运行
    
    ```
    # 查看服务运行状态
    systemctl status NetworkManager networking systemd-networkd
    # 关闭并禁用冗余服务，例如关闭传统networking
    sudo systemctl disable --now networking
    ```
    
2. **resolv.conf 软链接异常修复**
    
    90% 的配置不生效源于软链接错误，执行以下命令查看：
    
    bash
    
    运行
    
    ```
    ls -l /etc/resolv.conf
    ```
    
    正常输出应为软链接（指向 systemd-resolved/NetworkManager 的管理文件），若为普通文件，按对应方案重新创建软链接，或执行以下命令重置：
    
    bash
    
    运行
    
    ```
    # systemd-resolved方案重置
    sudo mv /etc/resolv.conf /etc/resolv.conf.bak
    sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
    sudo systemctl restart systemd-resolved
    ```
    
3. **DNS 缓存刷新**
    
    修改后解析仍用旧记录，刷新本地 DNS 缓存即可：
    
    bash
    
    运行
    
    ```
    # systemd-resolved 刷新缓存
    sudo resolvectl flush-caches
    # 查看缓存统计
    resolvectl statistics
    # NetworkManager 重载配置
    sudo nmcli general reload
    ```
    
4. **端口与防火墙排查**
    
    DNS 使用 53 端口 UDP/TCP，防火墙拦截会导致解析失败：
    
    bash
    
    运行
    
    ```
    # 测试DNS服务器连通性
    telnet 223.5.5.5 53
    nc -zv 8.8.8.8 53
    # Fedora firewalld放行DNS
    sudo firewall-cmd --add-service=dns --permanent
    sudo firewall-cmd --reload
    # Debian ufw放行DNS
    sudo ufw allow 53/tcp
    sudo ufw allow 53/udp
    sudo ufw reload
    ```
    
5. **SELinux 权限异常（仅 Fedora）**
    
    Fedora 默认开启 SELinux，修改软链接后权限异常可执行修复：
    
    bash
    
    运行
    
    ```
    sudo restorecon /etc/resolv.conf
    ```
    

---

## 四、补充资料

### 常用公共 DNS 推荐

表格

|类型|DNS 服务商|首选地址|备用地址|
|---|---|---|---|
|国内|阿里云公共 DNS|223.5.5.5|223.6.6.6|
|国内|腾讯云 DNSPod|119.29.29.29|182.254.116.116|
|国内|114DNS|114.114.114.114|114.114.115.115|
|国内|百度 DNS|180.76.76.76|-|
|国际|Google DNS|8.8.8.8|8.8.4.4|
|国际|Cloudflare DNS|1.1.1.1|1.0.0.1|
|国际|Quad9 安全 DNS|9.9.9.9|149.112.112.112|

### 快速速查表

表格

|场景|Fedora 首选方案|Debian 首选方案|
|---|---|---|
|临时测试|直接修改 /etc/resolv.conf|直接修改 /etc/resolv.conf|
|桌面环境|nmcli 命令 / 图形界面|nmcli 命令 / 图形界面|
|物理服务器|systemd-resolved 全局配置|/etc/network/interfaces|
|云服务器|nmcli/systemd-resolved|netplan|
|极简环境|systemd-networkd|systemd-resolved|