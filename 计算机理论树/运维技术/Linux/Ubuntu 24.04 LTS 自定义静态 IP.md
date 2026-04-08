本文档覆盖 Ubuntu 24.04 LTS（Noble Numbat）**桌面 / 服务器 / 云服务器**全场景，包含临时 / 永久修改、DNS 配套配置、生效验证、故障排查全流程，所有命令可直接复制使用，新手友好且兼容专业运维场景。

---

## 前置核心说明（必看，90% 配置失效问题源于此）

### 1. Ubuntu 24.04 默认网络管理规则

|系统版本|默认网络管理工具|配置入口|适用场景|
|---|---|---|---|
|Ubuntu 24.04 桌面版|NetworkManager|图形设置 /`nmcli`命令|本地桌面、带图形环境的工作站|
|Ubuntu 24.04 服务器版|Netplan（后端默认 systemd-networkd）|`/etc/netplan/*.yaml`|物理服务器、虚拟机、云服务器|

> ⚠️ 核心禁令：**禁止同时启用多个网络管理服务**（如同时开 NetworkManager+systemd-networkd），会导致配置冲突、IP 不生效、网络断连。

### 2. 前置必备查询命令（修改前后必须执行）

```bash
# 1. 查看网卡名称（核心！Ubuntu 24.04默认网卡名为ensXX/enp0sXX，无eth0）
ip addr

# 2. 查看默认网关地址（静态IP必须和网关同网段，否则无法上网）
ip route show default

# 3. 查看当前运行的网络管理服务（确认使用的工具）
systemctl status NetworkManager systemd-networkd

# 4. 查看当前DNS配置
resolvectl status | grep "DNS Server"
```

### 3. 关键安全与规范注意事项

1. **远程 SSH 操作必看**：修改服务器静态 IP 前，必须先用`netplan try`测试配置，严禁直接重启网络服务，避免配置错误导致远程断连。
2. **Netplan 语法铁律**：yaml 配置文件**严格使用 2 个空格缩进，禁止用 Tab 键**，冒号后必须加空格，否则直接报错。
3. **网段匹配规则**：自定义静态 IP 必须与网关在同一网段，例如网关`192.168.1.1`，IP 必须为`192.168.1.XXX`，子网掩码默认`/24`（对应 255.255.255.0）。
4. **云服务器特殊规则**：阿里云 / 腾讯云 / 华为云等云厂商 Ubuntu 24.04 镜像默认预装`cloud-init`，会自动覆盖网络配置，需先禁用其网络管理能力（文末有解决方案）。

---

## 一、临时修改 IP 地址（重启网络 / 系统失效，仅临时测试用）

仅适用于临时调试场景，配置立即生效，重启网卡 / 系统后自动还原，不会修改系统永久配置。

### 1. 临时修改 IP 与子网掩码

bash

运行

```
# 语法：sudo ip addr add 自定义IP/子网掩码CIDR dev 网卡名称
# 示例：给ens33网卡配置临时IP 192.168.1.100，子网掩码255.255.255.0（/24）
sudo ip addr add 192.168.1.100/24 dev ens33

# 可选：删除原有IP（避免冲突）
sudo ip addr del 原有IP/24 dev ens33
```

### 2. 临时配置默认网关

bash

运行

```
# 语法：sudo ip route add default via 网关地址 dev 网卡名称
sudo ip route add default via 192.168.1.1 dev ens33
```

### 3. 临时配置 DNS 服务器

bash

运行

```
# 临时修改resolv.conf，重启后失效
sudo vi /etc/resolv.conf
```

写入以下内容（多个 DNS 按优先级排序）：

conf

```
nameserver 223.5.5.5
nameserver 8.8.8.8
```

保存退出后立即生效。

---

## 二、永久修改静态 IP 地址（按场景优先级排序）

### 方案 1：图形界面修改（Ubuntu 24.04 桌面版首选，新手零门槛）

适合本地桌面环境，可视化操作无命令门槛，零出错率。

1. 打开系统 `设置 -> 网络`，找到目标网卡（有线 / Wi-Fi），点击右侧齿轮图标进入配置页。
2. 切换到 `IPv4` 选项卡，将 `IPv4方式` 从「自动 (DHCP)」改为「手动」。
3. 在「地址」栏点击「添加」，填写 3 项核心配置：
    
    - 地址：你的自定义静态 IP（如`192.168.1.100`）
    - 子网掩码：默认`255.255.255.0`（对应 CIDR /24）
    - 网关：你的路由器 / 交换机网关地址（如`192.168.1.1`）
    
4. 在「DNS」栏关闭「自动」开关，填写 DNS 服务器地址，多个用英文逗号分隔（推荐国内公共 DNS：`223.5.5.5,8.8.8.8`）。
5. 点击右上角「应用」保存配置，关闭对应网络开关再重新打开（或重启系统），配置即可永久生效。
6. 验证：打开终端执行`ip addr`，查看网卡 IP 是否为自定义地址。

### 方案 2：nmcli 命令行修改（桌面 / 服务器通用，NetworkManager 环境首选）

适配 Ubuntu 24.04 桌面版默认环境，也适用于安装了 NetworkManager 的服务器，稳定性最高，配置不被系统覆盖。

#### 完整操作步骤

1. 查看当前网络连接名称（核心！记住`NAME`列的名称，含空格需加引号）
    
    bash
    
    运行
    
    ```
    nmcli connection show
    ```
    
    示例输出：
    
    plaintext
    
    ```
    NAME                UUID                                  TYPE      DEVICE
    Wired connection 1  xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  ethernet  ens33
    ```
    
2. 执行命令修改静态 IP 配置（替换为你的连接名和网卡信息）
    
    bash
    
    运行
    
    ```
    # 1. 配置IPv4静态IP、子网掩码、网关（多个IP用逗号分隔）
    sudo nmcli connection modify "Wired connection 1" ipv4.addresses "192.168.1.100/24"
    sudo nmcli connection modify "Wired connection 1" ipv4.gateway "192.168.1.1"
    
    # 2. 配置DNS服务器，多个地址用英文逗号分隔
    sudo nmcli connection modify "Wired connection 1" ipv4.dns "223.5.5.5,8.8.8.8"
    
    # 3. 关闭DHCP自动获取，改为静态IP模式（核心！否则配置会被DHCP覆盖）
    sudo nmcli connection modify "Wired connection 1" ipv4.method manual
    
    # 4. 禁用IPv6（可选，无需IPv6可关闭，避免干扰）
    sudo nmcli connection modify "Wired connection 1" ipv6.method ignore
    ```
    
3. 重启连接使配置永久生效
    
    bash
    
    运行
    
    ```
    sudo nmcli connection up "Wired connection 1"
    ```
    
4. 验证生效
    
    bash
    
    运行
    
    ```
    # 查看IP配置
    ip addr show ens33
    # 查看DNS配置
    nmcli device show ens33 | grep DNS
    # 测试网络连通性
    ping -c 3 baidu.com
    ```
    

### 方案 3：Netplan 方案（Ubuntu 24.04 服务器版默认首选，云服务器通用）

Ubuntu 24.04 服务器版、云服务器镜像默认使用 Netplan 管理网络，是官方推荐的服务器网络配置方案，配置简洁、兼容性强。

#### 完整操作步骤

1. 查看并备份原有 Netplan 配置文件
    
    bash
    
    运行
    
    ```
    # 1. 查看配置文件（文件名通常为00-installer-config.yaml/50-cloud-init.yaml）
    ls /etc/netplan/
    
    # 2. 备份原配置文件（出错可直接回滚）
    sudo cp /etc/netplan/00-installer-config.yaml /etc/netplan/00-installer-config.yaml.bak
    ```
    
2. 编辑配置文件，写入静态 IP 配置
    
    bash
    
    运行
    
    ```
    sudo vi /etc/netplan/00-installer-config.yaml
    ```
    
    根据你的场景选择对应配置，**严格遵循 2 空格缩进，禁止用 Tab**：
    
    ##### 场景 1：单网卡静态 IP（最常用，服务器 / 虚拟机通用）
    
    yaml
    
    ```
    network:
      version: 2
      renderer: networkd  # 服务器默认用networkd，桌面版可改为NetworkManager
      ethernets:
        ens33:  # 替换为你的网卡名称，通过ip addr查询
          addresses:
            - 192.168.1.100/24  # 自定义静态IP/子网掩码CIDR
          routes:
            - to: default
              via: 192.168.1.1  # 默认网关地址
          nameservers:
            addresses: [223.5.5.5, 8.8.8.8]  # DNS服务器地址
          dhcp4: no  # 关闭DHCP自动获取，核心！
    ```
    
    ##### 场景 2：单网卡双 IP（同网卡配置多个静态 IP）
    
    yaml
    
    ```
    network:
      version: 2
      renderer: networkd
      ethernets:
        ens33:
          addresses:
            - 192.168.1.100/24
            - 192.168.1.101/24  # 第二个IP
          routes:
            - to: default
              via: 192.168.1.1
          nameservers:
            addresses: [223.5.5.5, 8.8.8.8]
          dhcp4: no
    ```
    
    ##### 场景 3：双网卡内外网分离（内网 + 外网双网卡）
    
    yaml
    
    ```
    network:
      version: 2
      renderer: networkd
      ethernets:
        ens33:  # 外网网卡
          addresses:
            - 192.168.1.100/24
          routes:
            - to: default
              via: 192.168.1.1  # 外网网关
          nameservers:
            addresses: [223.5.5.5, 8.8.8.8]
          dhcp4: no
        ens34:  # 内网网卡
          addresses:
            - 10.0.0.100/24
          routes:
            - to: 10.0.0.0/8
              via: 10.0.0.1  # 内网网关
          dhcp4: no
    ```
    
3. 测试配置（远程 SSH 操作**必须先执行此步**，配置错误会自动回滚，避免断连）
    
    bash
    
    运行
    
    ```
    sudo netplan try
    ```
    
    执行后按回车确认，若配置有语法错误，会自动提示并在 120 秒后回滚到原有配置，不会导致远程断连。
    
4. 正式应用配置，永久生效
    
    bash
    
    运行
    
    ```
    sudo netplan apply
    ```
    
5. 验证生效
    
    bash
    
    运行
    
    ```
    # 查看IP配置
    ip addr
    # 查看路由网关
    ip route show default
    # 测试网络连通性
    ping -c 3 baidu.com
    ```
    

### 方案 4：传统 ifupdown 方案（/etc/network/interfaces，兼容老版本）

Ubuntu 24.04 默认不预装，适合从老版本 Ubuntu 升级、习惯传统配置方式的用户，兼容性强。

1. 安装依赖包
    
    bash
    
    运行
    
    ```
    sudo apt update && sudo apt install -y ifupdown resolvconf
    ```
    
2. 禁用其他网络管理服务（避免冲突）
    
    bash
    
    运行
    
    ```
    sudo systemctl disable --now NetworkManager systemd-networkd
    ```
    
3. 编辑传统网络配置文件
    
    bash
    
    运行
    
    ```
    sudo vi /etc/network/interfaces
    ```
    
    写入静态 IP 配置（替换为你的网卡信息）：
    
    conf
    
    ```
    # 环回地址配置，无需修改
    auto lo
    iface lo inet loopback
    
    # 静态IP配置
    auto ens33  # 替换为你的网卡名称
    iface ens33 inet static
        address 192.168.1.100/24  # 静态IP/子网掩码
        gateway 192.168.1.1        # 默认网关
        dns-nameservers 223.5.5.5 8.8.8.8  # DNS服务器，多个用空格分隔
    ```
    
4. 重启网络服务使配置生效
    
    bash
    
    运行
    
    ```
    sudo systemctl restart networking
    # 或单网卡重启（推荐，不影响其他网卡）
    sudo ifdown ens33 && sudo ifup ens33
    ```
    
5. 验证生效：`ip addr` 查看 IP 配置。
    

---

## 三、云服务器专属配置（解决 cloud-init 覆盖网络配置问题）

阿里云、腾讯云、华为云等云厂商的 Ubuntu 24.04 镜像，默认预装`cloud-init`，会在系统重启后自动覆盖 Netplan 配置，导致自定义 IP 失效，需按以下步骤禁用其网络管理能力：

1. 编辑 cloud-init 主配置文件
    
    bash
    
    运行
    
    ```
    sudo vi /etc/cloud/cloud.cfg
    ```
    
2. 在文件末尾添加以下配置，禁用网络自动配置
    
    cfg
    
    ```
    network:
      config: disabled
    ```
    
3. 保存退出后，重新执行`netplan apply`应用静态 IP 配置，后续重启系统不会再被覆盖。

---

## 四、DNS 配套配置与优化

Ubuntu 24.04 默认通过`systemd-resolved`管理 DNS，自定义 IP 时需同步配置 DNS，否则会出现 “能 ping 通网关但上不了网” 的问题。

### 1. 全局 DNS 永久配置（适配所有网络管理方案）

bash

运行

```
# 编辑systemd-resolved配置文件
sudo vi /etc/systemd/resolved.conf
```

找到`[Resolve]`段，取消注释并修改配置：

conf

```
[Resolve]
DNS=223.5.5.5 8.8.8.8  # 主DNS，多个用空格分隔
FallbackDNS=114.114.114.114 1.1.1.1  # 备用DNS
Domains=local
DNSSEC=allow-downgrade
# 可选：开启DNS over TLS加密
DNSOverTLS=opportunistic
```

重启服务生效：

bash

运行

```
sudo systemctl restart systemd-resolved
```

### 2. 常用公共 DNS 推荐

表格

|服务商|首选 IPv4 地址|备用 IPv4 地址|特点|
|---|---|---|---|
|阿里云|223.5.5.5|223.6.6.6|国内访问速度快，稳定性高|
|腾讯云 DNSPod|119.29.29.29|182.254.116.116|低延迟，防污染|
|114DNS|114.114.114.114|114.114.115.115|国内老牌，兼容性强|
|Google|8.8.8.8|8.8.4.4|国际通用，无劫持|
|Cloudflare|1.1.1.1|1.0.0.1|隐私性强，全球节点多|

---

## 五、故障排查指南（配置不生效 / 无法上网必看）

### 1. 配置不生效，IP 未变更

- 排查 1：确认网络管理服务无冲突，仅保留一个服务运行，关闭其他冗余服务
    
    bash
    
    运行
    
    ```
    # 示例：关闭NetworkManager，仅保留systemd-networkd
    sudo systemctl disable --now NetworkManager
    ```
    
- 排查 2：Netplan 配置检查缩进和语法，执行`sudo netplan generate`检测语法错误
- 排查 3：云服务器检查是否禁用了 cloud-init 的网络管理，未禁用会导致重启后配置被覆盖

### 2. 能 ping 通网关，无法访问外网

- 排查 1：DNS 配置错误，执行`nslookup baidu.com`检测 DNS 解析是否正常，重新配置 DNS
- 排查 2：网关配置错误，执行`ip route show default`确认网关地址正确，且能 ping 通网关
- 排查 3：防火墙拦截，执行`sudo ufw status`查看防火墙规则，临时关闭测试`sudo ufw disable`

### 3. 静态 IP 配置后，仍获取到 DHCP 地址

- 核心原因：未关闭 DHCP 自动获取，检查配置中是否设置`dhcp4: no`（Netplan）或`ipv4.method manual`（nmcli）
- 补充：关闭系统自带的 dhcpcd 服务`sudo systemctl disable --now dhcpcd`

### 4. 网卡未启动，IP 不生效

- 执行`sudo ip link set 网卡名称 up`启用网卡
- 查看网卡状态`ip link show 网卡名称`，确认状态为`UP`

---

## 六、常用 CIDR 子网掩码对照表（新手速查）

表格

|CIDR 格式|子网掩码|可用 IP 数量|适用场景|
|---|---|---|---|
|/24|255.255.255.0|254|家用、小型局域网（最常用）|
|/16|255.255.0.0|65534|中大型局域网|
|/8|255.0.0.0|16777214|超大型网络|
|/27|255.255.255.224|30|小型 VLAN、服务器网段|