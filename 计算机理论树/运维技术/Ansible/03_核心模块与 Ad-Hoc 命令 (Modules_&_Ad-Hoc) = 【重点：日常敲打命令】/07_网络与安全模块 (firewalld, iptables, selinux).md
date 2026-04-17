
>网络与安全模块是 Ansible 管理被控节点防火墙与安全策略的核心工具，覆盖 firewalld、iptables、SELinux 三大 Linux 安全体系，所有模块均 **内置完整幂等性** 。

### 3.7.1 firewalld 模块

**核心定义**：用于管理 RHEL/CentOS/Fedora 等系统的 firewalld 防火墙规则，支持永久配置、运行时配置、区域、端口、服务、富规则等全功能管理。

#### 核心参数

|参数|核心作用|必填项|
|:--|:--|:--|
|`zone`|防火墙区域，如 public、trusted、internal，默认 public|是|
|`service`|要管理的服务名称，如 http、ssh、mysql|与 port 二选一|
|`port`|要管理的端口 / 协议，格式为 port/protocol，如 80/tcp|与 service 二选一|
|`state`|规则状态，enabled（允许）、disabled（禁止）|是|
|`permanent`|是否永久写入配置，默认 no，仅运行时生效|否|
|`immediate`|是否立即生效，默认 no|否|
|`source`|源地址 / 网段，用于富规则与源地址绑定|否|
|`rich_rule`|firewalld 富规则字符串|否|
|`reload`|修改永久配置后是否重载 firewalld，默认 no|否|

#### 标准示例

```bash
# 1. 永久允许http服务，立即生效
ansible all -m firewalld -a "zone=public service=http permanent=yes state=enabled immediate=yes" -b

# 2. 永久允许8080/tcp端口，重载生效
ansible all -m firewalld -a "zone=public port=8080/tcp permanent=yes state=enabled reload=yes" -b

# 3. 允许指定网段访问ssh端口
ansible all -m firewalld -a "zone=public rich_rule='rule family=ipv4 source address=192.168.1.0/24 service name=ssh accept' permanent=yes state=enabled" -b
```

#### 核心注意事项

1. 必须提权执行，内置幂等性；
2. 高频踩坑点：`permanent=yes`仅写入永久配置文件，不会立即生效，必须配合`immediate=yes`或`reload=yes`才能生效；
3. 生产环境操作防火墙需谨慎，避免误封 SSH 端口，导致被控节点失联，建议先通过`-C`参数干跑验证。

### 3.7.2 iptables 模块

**核心定义**：用于管理 Linux 系统的 iptables/netfilter 防火墙规则，支持表、链、规则、动作、地址 / 端口匹配等全功能配置，适配无 firewalld 的系统场景。

#### 核心参数

|参数|核心作用|必填项|
|:--|:--|:--|
|`table`|iptables 表，filter（默认）、nat、mangle、raw|否|
|`chain`|链名称，filter 表默认 INPUT，nat 表可选 PREROUTING/POSTROUTING|否|
|`protocol`|协议，tcp（默认）、udp、icmp、all|否|
|`source`|源地址 / 网段|否|
|`dport`|目标端口，支持范围|否|
|`jump`|规则匹配后的动作，ACCEPT、DROP、REJECT、DNAT、SNAT 等|是|
|`state`|规则状态，present（默认）、absent|否|
|`ctstate`|连接跟踪状态，NEW、ESTABLISHED、RELATED|否|
|`comment`|规则注释|否|

#### 标准示例

```bash
# 1. 允许80/tcp端口入站
ansible all -m iptables -a "chain=INPUT protocol=tcp dport=80 ctstate=NEW jump=ACCEPT comment='Allow HTTP'" -b

# 2. 允许指定网段访问22端口
ansible all -m iptables -a "chain=INPUT protocol=tcp source=192.168.1.0/24 dport=22 jump=ACCEPT" -b

# 3. 配置NAT端口转发，80端口转发到8080
ansible all -m iptables -a "table=nat chain=PREROUTING protocol=tcp dport=80 jump=DNAT to_destination=127.0.0.1:8080" -b

# 4. 删除指定规则
ansible all -m iptables -a "chain=INPUT protocol=tcp dport=80 jump=ACCEPT state=absent" -b
```

#### 核心注意事项

1. 必须提权执行，内置幂等性，基于规则全参数匹配，避免重复规则；
2. iptables 规则按顺序匹配，Ansible 默认将规则添加到链的顶部，可通过 position 参数指定位置；
3. 规则仅运行时生效，重启后丢失，需配合 iptables-save 命令永久保存；
4. 生产环境操作需极度谨慎，避免误封 SSH 端口导致失联。

### 3.7.3 selinux 模块

**核心定义**：用于管理 RHEL/CentOS 等系统的 SELinux 安全模块，包括全局模式设置、策略配置等。

#### 核心参数与标准示例

核心参数包括`state`（enforcing/permissive/disabled，必填）、`policy`（策略类型，默认 targeted）、`update_kernel_param`（更新内核参数，禁用 SELinux 时需设置为 yes）。

```bash
# 1. 设置SELinux为宽容模式，永久生效
ansible all -m selinux -a "state=permissive policy=targeted" -b

# 2. 启用SELinux强制模式
ansible all -m selinux -a "state=enforcing policy=targeted" -b

# 3. 完全禁用SELinux，更新内核参数
ansible all -m selinux -a "state=disabled update_kernel_param=yes" -b
```

#### 核心注意事项

1. 必须提权执行，内置幂等性；
2. `state=disabled`需要重启系统才能完全生效，运行时仅能切换 enforcing/permissive 模式；
3. 生产环境不建议直接禁用 SELinux，推荐使用 enforcing 模式配合正确的策略配置，提升系统安全性。

---
