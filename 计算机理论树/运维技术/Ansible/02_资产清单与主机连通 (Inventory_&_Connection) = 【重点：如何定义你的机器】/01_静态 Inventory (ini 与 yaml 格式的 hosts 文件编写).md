## 本章学习目标

本章完整覆盖 Ansible 被控节点管理的核心载体 ——Inventory 资产清单的全体系知识，建立 Ansible 对基础设施的管理边界认知。内容从静态清单的标准化编写、主机分组逻辑、内置连接参数，到 SSH 免密登录的原理与批量配置、企业级动态 Inventory 实现，最终完成连通性验证与底层原理解析，无核心知识点遗漏，是后续所有 Ad-Hoc 命令与 Playbook 执行的前置核心基础。

---

## 2.1 Inventory 核心定义与定位

Inventory（资产清单 / 主机清单）是 Ansible 用于定义、管理和组织所有被控节点的核心载体，是控制节点与被控节点之间的映射桥梁，Ansible 所有自动化操作的目标范围、连接规则、节点元数据、层级变量均通过 Inventory 定义。

其核心职能包括 4 个维度：

1. **范围定义**：明确 Ansible 可管理的被控节点集合，支持主机名、IP 地址、域名等标识形式；
2. **逻辑分组**：对被控节点进行多维度分组，实现批量、分层级的自动化管理；
3. **连接配置**：定义节点专属的远程连接、认证、权限、环境参数，适配异构基础设施；
4. **元数据承载**：承载主机级、组级变量，为自动化任务提供业务、环境、配置等元数据支撑。

Ansible Inventory 分为两大核心类型：**静态 Inventory**（固定文本文件，手动维护）与**动态 Inventory**（可执行程序 / 插件，自动拉取生成），可独立使用，也可混合部署。

---

## 2.2 静态 Inventory

静态 Inventory 是 Ansible 默认支持、最常用的主机清单形式，为固定格式的文本文件，支持 INI 与 YAML 两种标准化语法，适用于主机数量固定、拓扑变化频率低的场景。

### 2.2.1 内置默认组

Ansible 有两个强制内置的全局默认组，所有定义在 Inventory 中的主机都会自动归属，不可手动删除或修改：

1. **all 组**：全局最大组，包含 Inventory 中定义的所有主机，无论是否归属自定义分组，是 Ansible 全局批量操作的核心目标；
2. **ungrouped 组**：未归属任何自定义分组的主机，自动归入该组，是除 all 组外唯一的默认组。

### 2.2.2 INI 格式静态 Inventory

INI 格式是 Ansible 默认的 Inventory 格式，语法极简、可读性强，上手门槛低，适用于绝大多数简单场景。其核心语法规则如下：

- 以行为单位，`#` 与 `;` 为注释符，支持行首与行尾注释；
- 用 `[组名]` 定义主机分组，组名下为该组的主机列表；
- 支持主机名、IPv4/IPv6 地址、FQDN 域名，支持范围简写批量定义主机；
- 支持行内直接定义主机级内置参数与变量。

#### 基础语法示例

```ini
# 注释：未分组主机，自动归属all与ungrouped组
192.168.1.100
monitor01.example.com

# 定义Web服务器分组
[webservers]
192.168.1.101
192.168.1.102
# 带自定义端口的主机定义
web03.example.com ansible_port=2222
# 主机别名定义：用web04作为别名，实际连接地址为192.168.1.104
web04 ansible_host=192.168.1.104

# 定义数据库服务器分组
[dbservers]
192.168.1.201
192.168.1.202
```

#### 主机范围简写语法

支持数字与字母范围匹配，批量定义连续命名的主机，大幅简化大规模集群的清单编写，语法为 `[起始值:结束值]`，支持前导零：

```ini
# 匹配192.168.1.101 ~ 192.168.1.110 共10台主机
[webservers]
192.168.1.[101:110]

# 匹配web01 ~ web10 共10台主机，支持前导零
[webservers]
web[01:10].example.com

# 匹配a~z的主机名
[test_nodes]
node[a:z].example.com
```

#### 组级变量定义

通过 `[组名:vars]` 定义组级变量，组内所有主机自动继承该变量，无需单主机重复配置：

```ini
[webservers:vars]
ansible_user=ops
ansible_ssh_private_key_file=/home/ops/.ssh/id_ed25519
ansible_become=yes
ansible_python_interpreter=/usr/bin/python3
nginx_version=1.24.0
```

### 2.2.3 YAML 格式静态 Inventory

YAML 格式支持更复杂的层级结构、嵌套分组与变量定义，语法严格遵循 YAML 规范（缩进敏感、大小写敏感），适用于主机规模大、分组层级复杂、变量多的企业级场景。

#### 基础语法示例

与上述 INI 格式完全对等的 YAML 结构如下：

```yaml
---
all:
  hosts:
    # 未分组主机，自动归属ungrouped组
    192.168.1.100:
    monitor01.example.com:
  children:
    # Web服务器子组
    webservers:
      hosts:
        192.168.1.101:
        192.168.1.102:
        web03.example.com:
          ansible_port: 2222
        web04:
          ansible_host: 192.168.1.104
      # 组级变量定义
      vars:
        ansible_user: ops
        ansible_ssh_private_key_file: /home/ops/.ssh/id_ed25519
        ansible_become: yes
        ansible_python_interpreter: /usr/bin/python3
        nginx_version: 1.24.0
    # 数据库服务器子组
    dbservers:
      hosts:
        192.168.1.201:
        192.168.1.202:
```

YAML 格式同样支持范围简写语法，与 INI 格式完全一致，可直接在 hosts 字段中使用。

### 2.2.4 两种格式的选型适配

|格式|核心优势|核心局限|适配场景|
|:--|:--|:--|:--|
|INI 格式|语法极简、上手快、可读性强、编写效率高|复杂嵌套分组支持弱、大量变量定义结构混乱|中小规模集群、简单分组、快速测试场景|
|YAML 格式|层级结构清晰、支持复杂嵌套、变量定义规范、可扩展性强|语法严格、缩进敏感、编写门槛略高|大规模集群、多层级分组、企业级标准化场景|
