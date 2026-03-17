## 一、核心定位

`ip addr` 是 Linux 上**最核心、最标准的网络接口地址查看与管理命令**，属于 `iproute2` 软件包，是旧命令 `ifconfig` 的**现代替代品**，功能更强大、语法更统一。核心作用是**查看所有网络接口的 IPv4/IPv6 地址、MAC 地址、接口状态，添加 / 删除临时 IP 地址，配合 `ip link` 管理接口启用 / 禁用**，是网络配置、故障排查的必备基础命令，也是 Linux 入门必学的网络工具。

> 补充说明：`ip addr` 可缩写为 `ip a`，是最常用的简写；与 `ifconfig` 不同，`ip addr` 仅显示地址信息，接口状态管理用 `ip link`，路由管理用 `ip route`，三者并称为 `iproute2` 三剑客；信息来源于 `/proc/net/dev`、`/proc/net/if_inet6` 等文件。

## 二、语法格式

```bash
# 核心语法：查看/管理网络接口地址
ip addr [子命令] [选项]
```

> 关键语法说明：
> 
> 1. **子命令**：如 `show`（查看）、`add`（添加）、`del`（删除）等；
> 2. **常用简写**：`ip addr show` 可简写为 `ip a`，`ip addr show eth0` 可简写为 `ip a eth0`；
> 3. **默认行为**：无参数时默认执行 `ip addr show`，查看所有接口的地址信息。

## 三、高频核心命令

### 1. 查看类（最常用）

|命令|核心作用|高频使用场景|
|:--|:--|:--|
|`ip addr show` / `ip a`|查看**所有网络接口**的地址信息（IPv4/IPv6、MAC、状态、MTU 等）|快速查看所有网络接口，最基础用法|
|`ip addr show eth0` / `ip a eth0`|查看**指定接口**（如 `eth0`、`ens33`、`wlan0`）的地址信息|查看特定接口的详细配置|
|`ip addr show up` / `ip a up`|仅查看**已启用**（UP）的网络接口|专注查看活跃接口|
|`ip addr show scope global`|仅查看**全局作用域**的 IP 地址（公网 / 局域网地址）|查看可对外访问的地址|

### 2. 地址管理类（临时，重启失效）

|命令|核心作用|高频使用场景|
|:--|:--|:--|
|`ip addr add 192.168.1.10/24 dev eth0`|给指定接口**添加临时 IPv4 地址**（`/24` 是 CIDR 格式的子网掩码，必须加）|临时配置接口地址，测试用|
|`ip addr add 2001:db8::1/64 dev eth0`|给指定接口**添加临时 IPv6 地址**|临时配置 IPv6 地址|
|`ip addr del 192.168.1.10/24 dev eth0`|删除指定接口的**临时 IPv4 地址**|移除临时配置的地址|
|`ip addr flush dev eth0`|清空指定接口的**所有临时 IP 地址**|快速清除接口的所有临时地址|

### 3. 配合 `ip link` 管理接口状态

|命令|核心作用|高频使用场景|
|:--|:--|:--|
|`ip link set eth0 up`|启用指定网络接口|激活接口|
|`ip link set eth0 down`|禁用指定网络接口|停用接口|
|`ip link set eth0 mtu 1500`|设置指定接口的**MTU**（最大传输单元）|调整接口 MTU|

## 四、核心列含义说明（`ip addr show` 输出）

|区域 / 列|含义|说明|
|:--|:--|:--|
|**接口名**|`lo`|回环接口（loopback），用于本地通信，地址 `127.0.0.1`/`::1`|
||`eth0`/`ens33`/`enp0s3`|物理以太网接口（命名规则随发行版 / 驱动变化）|
||`wlan0`/`wlp2s0`|无线局域网接口|
|**接口状态**|`UP`|接口已启用|
||`DOWN`|接口已禁用|
||`mtu 1500`|最大传输单元为 1500 字节（默认）|
|**MAC 地址**|`link/ether 00:11:22:33:44:55`|接口的物理 MAC 地址|
|**IPv4 地址**|`inet 192.168.1.10/24 brd 192.168.1.255 scope global dynamic eth0`|IPv4 地址 / 子网掩码 / 广播地址 / 作用域 / 获取方式（`dynamic` 为 DHCP，`static` 为静态）/ 接口名|
|**IPv6 地址**|`inet6 2001:db8::1/64 scope global`|IPv6 地址 / 前缀 / 作用域|
||`inet6 fe80::211:22ff:fe33:4455/64 scope link`|IPv6 链路本地地址（`scope link`，仅本地网段通信）|

## 五、避坑提示与注意事项

1. **新手最高频踩坑：是 `ifconfig` 的替代品，不要混用，且临时添加重启失效**
    
    - `ip addr` 是**现代标准**，`ifconfig` 已废弃（多数发行版默认不安装）；
    - 强制要求：用 `ip addr`，不要用 `ifconfig`；
    - 典型错误：`ip addr add` 添加的地址重启后消失；
    - 正确示例：临时测试用 `ip addr add`，永久配置需修改网络配置文件（见下文）。
    
2. **添加地址必须加 CIDR 格式的子网掩码（/24），否则默认 /32**
    
    - CIDR 格式（如 `/24` 对应 `255.255.255.0`）是必须的，否则默认 `/32`（仅单个地址，无法通信）；
    - 强制要求：添加地址必须加 `/24`（或对应子网的 CIDR）；
    - 典型错误：`ip addr add 192.168.1.10 dev eth0`（默认 /32，无法通信）；
    - 正确示例：`ip addr add 192.168.1.10/24 dev eth0`。
    
3. **临时添加重启失效，永久配置需修改网络配置文件**
    
    - `ip addr add`/`del` 是**临时**的，重启网络或重启系统后失效；
    - 永久配置需修改对应发行版的网络配置文件：
        
        - **Debian/Ubuntu（旧）**：`/etc/network/interfaces`；
        - **Debian/Ubuntu（新，Netplan）**：`/etc/netplan/*.yaml`；
        - **RHEL/CentOS/Fedora**：`/etc/sysconfig/network-scripts/ifcfg-eth0`；
        
    - 典型错误：以为 `ip addr add` 是永久的；
    - 正确示例：临时测试用 `ip addr`，永久用配置文件。
    
4. **`ip link` 和 `ip addr` 配合使用，接口状态管理用 `ip link`**
    
    - `ip addr` 仅管理地址，接口启用 / 禁用、MTU 设置用 `ip link`；
    - 解决方法：两者配合使用；
    - 典型错误：用 `ip addr` 尝试启用接口；
    - 正确示例：`ip link set eth0 up` 启用接口，`ip addr add` 添加地址。
    

## 六、同类对比

表格

|命令|核心作用|与 `ip addr` 的区别|
|:--|:--|:--|
|`ifconfig`|旧的网络接口配置工具|`ip addr` 是现代标准，功能更强；`ifconfig` 已废弃，多数发行版默认不安装。推荐用 `ip addr`，不要用 `ifconfig`。|
|`ip link`|管理网络接口状态（启用 / 禁用、MTU、MAC）|`ip addr` 管理地址；`ip link` 管理接口状态。两者配合使用。|
|`ip route`|管理路由表|`ip addr` 管理接口地址；`ip route` 管理路由。两者配合使用。|

## 七、课后练习

1. **基础查看练习**：用 `ip addr show` 或 `ip a` 查看所有网络接口的地址信息，理解各列的含义（接口名、状态、MAC、IPv4/IPv6、作用域）。
2. **指定接口查看练习**：用 `ip addr show eth0`（替换为你的接口名）查看指定接口的详细配置。
3. **仅启用接口查看练习**：用 `ip addr show up` 仅查看已启用的接口。
4. **临时添加地址练习**：用 `ip addr add 192.168.100.10/24 dev eth0`（替换为你的接口名）添加临时 IPv4 地址，用 `ip a` 验证，然后重启网络（`systemctl restart NetworkManager` 或 `systemctl restart networking`），验证地址是否消失。
5. **删除地址练习**：重新添加临时地址后，用 `ip addr del 192.168.100.10/24 dev eth0` 删除，用 `ip a` 验证。
6. **清空地址练习**：添加多个临时地址后，用 `ip addr flush dev eth0` 清空所有临时地址，用 `ip a` 验证。
7. **配合 `ip link` 练习**：用 `ip link set eth0 down` 禁用接口，用 `ip a` 验证状态为 `DOWN`；再用 `ip link set eth0 up` 启用接口，验证状态为 `UP`。
8. **同类对比练习**：如果安装了 `ifconfig`（`sudo apt install net-tools` 或 `sudo yum install net-tools`），同时运行 `ip a` 和 `ifconfig`，对比两者的输出。