### 5.5.1 严格定义

魔法变量是 Ansible 引擎内置的特殊全局变量，**不是从被控节点采集的 Facts 信息**，而是由 Ansible 控制节点在执行过程中自动生成的，用于获取 Inventory 元数据、跨主机变量引用、执行上下文信息，是实现多节点协同、复杂编排的核心能力。

本节严格覆盖大纲指定的 3 个核心魔法变量：`hostvars`、`groups`、`inventory_hostname`，同时补充配套高频魔法变量。

### 5.5.2 核心魔法变量详解

#### 1. hostvars：跨主机变量引用

**核心定义**：一个全局字典变量，包含所有被控节点的所有变量（Facts、Inventory 变量、Register 变量、set_fact 变量等），可在任意节点的 Task、模板中，通过`hostvars['主机名']`引用其他节点的变量，实现跨主机的数据协同。

**核心使用场景**：

- 多节点集群配置，如 Web 节点引用数据库节点的 IP 地址；
- 跨 Play 变量传递，在后续 Play 中引用前序 Play 生成的变量；
- 集群配置自动发现，动态生成集群节点列表。

**标准示例**：

```yaml
---
- name: 数据库节点Play，生成数据库IP变量
  hosts: dbservers
  tasks:
    - name: 设置数据库IP变量
      set_fact:
        db_server_ip: "{{ ansible_default_ipv4.address }}"

- name: Web节点Play，引用数据库节点的变量
  hosts: webservers
  tasks:
    - name: 输出数据库节点IP
      debug:
        msg: "数据库服务器地址：{{ hostvars['192.168.1.201'].db_server_ip }}"

    - name: 生成Web应用配置
      template:
        src: app.conf.j2
        dest: /opt/app/app.conf
```

模板文件`app.conf.j2`中跨主机引用：

```jinja2
# 应用配置文件
DB_HOST = {{ hostvars[groups['dbservers'][0]].db_server_ip }}
DB_PORT = 3306
```

#### 2. groups：Inventory 分组信息获取

**核心定义**：一个全局字典变量，key 为 Inventory 中的分组名，value 为该分组下的所有主机名列表，可获取 Inventory 的全部分组结构与主机列表，实现动态遍历分组内的所有节点。

**核心使用场景**：

- 动态生成集群节点列表，如 Nginx upstream、Redis 集群、K8s 节点配置；
- 遍历分组内的所有主机，批量生成配置；
- 校验分组内的主机数量，实现集群规模校验。

**标准示例**：

```yaml
---
- name: groups变量示例
  hosts: webservers
  tasks:
    - name: 输出所有主机列表
      debug:
        msg: "Inventory中所有主机：{{ groups['all'] }}"

    - name: 输出数据库节点列表
      debug:
        msg: "数据库节点：{{ groups['dbservers'] }}"

    - name: 输出第一个数据库节点主机名
      debug:
        msg: "主数据库节点：{{ groups['dbservers'][0] }}"
```

Jinja2 模板中动态生成 Nginx upstream 配置：

```jinja2
# nginx_upstream.conf.j2
upstream backend {
    {% for host in groups['webservers'] %}
    server {{ hostvars[host].ansible_default_ipv4.address }}:80;
    {% endfor %}
}
```

#### 3. inventory_hostname：当前主机 Inventory 名称

**核心定义**：字符串变量，值为当前被控节点在 Inventory 中定义的主机名 / 别名 / IP 地址，是 Ansible 识别当前主机的唯一标识，与`ansible_hostname`（系统主机名）有本质区别。

**核心区别说明**：

|变量|来源|含义|不可用场景|
|:--|:--|:--|:--|
|`inventory_hostname`|Inventory 清单|主机在 Inventory 中定义的名称|无，执行前即可获取|
|`ansible_hostname`|Facts 采集|被控节点系统内的主机名|关闭 gather_facts 时无法使用|

**标准示例**：

```yaml
---
- name: inventory_hostname示例
  hosts: all
  gather_facts: no  # 关闭Facts采集，仍可使用inventory_hostname
  tasks:
    - name: 输出当前主机在Inventory中的名称
      debug:
        msg: "Inventory主机名：{{ inventory_hostname }}"

    - name: 生成主机专属配置文件
      template:
        src: host.conf.j2
        dest: "/opt/config/{{ inventory_hostname }}.conf"
```

### 5.5.3 补充高频魔法变量

|魔法变量名|核心含义|
|:--|:--|
|`group_names`|列表，当前主机所属的所有 Inventory 分组名|
|`inventory_dir`|字符串，当前使用的 Inventory 文件 / 目录的绝对路径|
|`inventory_file`|字符串，当前使用的 Inventory 主文件的绝对路径|
|`play_hosts`|列表，当前 Play 中匹配的所有主机名列表|
|`ansible_play_name`|字符串，当前 Play 的 name 字段值|

---
