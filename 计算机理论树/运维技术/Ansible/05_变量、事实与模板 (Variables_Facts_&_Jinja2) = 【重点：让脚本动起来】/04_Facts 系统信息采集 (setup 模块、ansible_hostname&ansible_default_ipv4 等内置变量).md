### 5.4.1 严格定义

Facts 是 Ansible 通过`setup`模块自动采集的被控节点的系统级信息，是一组内置的主机级变量，涵盖操作系统、硬件、网络、磁盘、内核、软件环境等全维度的节点元数据。Facts 在 Play 执行前默认自动采集，可直接在剧本、模板中引用，实现基于节点特性的动态适配。

### 5.4.2 Facts 采集核心机制

1. **采集触发**：Play 默认开启`gather_facts: yes`，执行 Play 的第一个动作就是通过 SSH 连接被控节点，执行`setup`模块，采集全量 Facts 信息；
2. **执行位置**：Facts 采集在被控节点本地执行，采集完成后返回控制节点，存储在对应主机的变量中；
3. **数据格式**：所有 Facts 均以`ansible_`为前缀，采用字典嵌套结构，支持层级引用。

### 5.4.3 Facts 查询与过滤

#### 1. 命令行查询全量 Facts

通过 Ad-Hoc 命令直接调用`setup`模块，查看指定主机的全量 Facts 信息：

```bash
# 查看指定主机的全量Facts
ansible 192.168.1.101 -m setup

# 将Facts输出到文件
ansible 192.168.1.101 -m setup > facts.json
```

#### 2. 过滤指定 Facts

通过`filter`参数过滤仅需的 Facts 字段，减少输出内容，提升采集效率：

```bash
# 仅过滤主机名相关Facts
ansible all -m setup -a "filter=ansible_hostname"

# 过滤操作系统相关Facts
ansible all -m setup -a "filter=ansible_os_*"

# 过滤默认网卡IPv4信息
ansible all -m setup -a "filter=ansible_default_ipv4"
```

### 5.4.4 生产环境高频常用 Facts

|Facts 变量名|核心含义|
|:--|:--|
|`ansible_hostname`|被控节点的系统主机名|
|`ansible_fqdn`|被控节点的全限定域名|
|`ansible_default_ipv4.address`|被控节点默认网卡的 IPv4 地址|
|`ansible_os_family`|操作系统家族，如 RedHat、Debian、Suse，用于跨发行版适配|
|`ansible_distribution`|操作系统发行版，如 CentOS、Ubuntu、Rocky Linux|
|`ansible_distribution_version`|操作系统发行版版本号|
|`ansible_kernel`|系统内核版本|
|`ansible_architecture`|系统架构，如 x86_64、aarch64|
|`ansible_processor_vcpus`|CPU 逻辑核心数|
|`ansible_memtotal_mb`|系统总内存大小（MB）|
|`ansible_mounts`|系统磁盘挂载点列表，包含容量、使用率等信息|
|`ansible_date_time`|系统当前时间信息，包含年、月、日、时间戳等|
|`ansible_python_version`|被控节点 Python 解释器版本|

### 5.4.5 核心实战场景

#### 场景 1：跨发行版适配

基于`ansible_os_family`实现不同操作系统的软件包管理适配，是跨环境剧本的核心能力：

```yaml
---
- name: 跨发行版安装Nginx
  hosts: all
  become: yes
  tasks:
    - name: RedHat系安装Nginx
      yum:
        name: nginx
        state: present
      when: ansible_os_family == "RedHat"

    - name: Debian系安装Nginx
      apt:
        name: nginx
        state: present
      when: ansible_os_family == "Debian"
```

#### 场景 2：动态生成配置文件

在 Jinja2 模板中引用 Facts 变量，动态生成适配节点特性的配置文件：

```jinja2
# nginx.conf.j2模板
worker_processes {{ ansible_processor_vcpus }};  # 自动适配CPU核心数
error_log /var/log/nginx/error.log;
events {
    worker_connections {{ ansible_memtotal_mb // 2 }};  # 基于内存动态设置连接数
}
```

#### 场景 3：条件判断与节点筛选

基于 Facts 信息筛选符合条件的节点，执行特定任务：

```yaml
---
- name: 仅对x86_64架构节点执行任务
  hosts: all
  tasks:
    - name: 安装x86_64专属软件
      yum:
        name: node_exporter
        state: present
      when: ansible_architecture == "x86_64"
```

---
