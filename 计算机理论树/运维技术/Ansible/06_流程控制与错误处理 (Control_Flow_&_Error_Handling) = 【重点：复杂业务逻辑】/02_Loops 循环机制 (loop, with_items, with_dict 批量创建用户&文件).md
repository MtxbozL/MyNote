### 6.2.1 严格定义

循环机制是 Ansible 实现批量同质化任务的核心能力，通过遍历可迭代对象（列表、字典等），用单个 Task 实现批量重复操作，避免冗余代码，大幅提升剧本的可维护性与简洁性。Ansible 提供两套循环体系：

- **标准 loop 关键字**：Ansible 2.5 + 官方推荐的通用循环方案，适配绝大多数场景；
- **with_* 系列循环**：传统兼容型循环，适配特定数据结构与场景，大纲指定的`with_items`、`with_dict`为最常用类型。

### 6.2.2 标准 loop 循环（官方推荐）

#### 1. 基础列表循环

最常用场景，遍历字符串 / 数值列表，批量执行相同操作，完美适配大纲指定的批量创建用户、文件等场景：

```yaml
---
- name: 基础列表循环批量创建用户
  hosts: all
  become: yes
  vars:
    # 待创建的用户列表
    system_users:
      - ops
      - dev
      - monitor
  tasks:
    - name: 批量创建系统用户
      user:
        name: "{{ item }}"
        state: present
        shell: /bin/bash
        create_home: yes
      loop: "{{ system_users }}"
```

> 核心说明：循环遍历过程中，当前迭代元素通过固定变量`item`引用，这是 loop 循环的核心规则。

#### 2. 字典列表循环

遍历包含多字段的字典列表，实现更复杂的批量配置，是生产环境的主流用法：

```yaml
---
- name: 字典列表循环批量创建目录
  hosts: webservers
  become: yes
  vars:
    app_dirs:
      - path: /data/app
        owner: nginx
        group: nginx
        mode: '0755'
      - path: /data/logs
        owner: nginx
        group: nginx
        mode: '0750'
      - path: /data/conf
        owner: root
        group: root
        mode: '0644'
  tasks:
    - name: 批量创建应用目录并配置权限
      file:
        path: "{{ item.path }}"
        state: directory
        owner: "{{ item.owner }}"
        group: "{{ item.group }}"
        mode: "{{ item.mode }}"
        recurse: yes
      loop: "{{ app_dirs }}"
```

#### 3. 循环 + 条件判断

循环中配合`when`语句，实现迭代元素的过滤，仅对符合条件的元素执行操作：

```yaml
---
- name: 循环过滤批量安装软件
  hosts: all
  become: yes
  vars:
    software_packages:
      - name: nginx
        install: yes
      - name: mysql
        install: no
      - name: redis
        install: yes
  tasks:
    - name: 批量安装标记为需要安装的软件
      yum:
        name: "{{ item.name }}"
        state: present
      loop: "{{ software_packages }}"
      when: item.install | bool
```

### 6.2.3 传统 with_* 系列循环

#### 1. with_items 循环

与基础 loop 列表循环功能完全对等，是 Ansible 2.5 之前的标准列表循环方式，目前仍广泛兼容：

```yaml
---
- name: with_items批量创建用户
  hosts: all
  become: yes
  tasks:
    - name: 批量创建系统用户
      user:
        name: "{{ item }}"
        state: present
        shell: /bin/bash
      with_items:
        - ops
        - dev
        - monitor
```

> 核心说明：`with_items`会自动扁平化嵌套列表，而标准 loop 默认不会，这是两者的核心行为差异。

#### 2. with_dict 循环

专门用于遍历字典的键值对，迭代元素通过`item.key`（键）和`item.value`（值）引用，适配字典类型的批量处理场景：

```yaml
---
- name: with_dict遍历字典配置系统内核参数
  hosts: all
  become: yes
  vars:
    sysctl_config:
      net.ipv4.ip_forward: 1
      net.ipv4.tcp_tw_reuse: 1
      net.ipv4.conf.all.accept_redirects: 0
  tasks:
    - name: 批量配置内核参数
      sysctl:
        name: "{{ item.key }}"
        value: "{{ item.value }}"
        state: present
        sysctl_file: /etc/sysctl.d/99-custom.conf
        reload: yes
      with_dict: "{{ sysctl_config }}"
```

### 6.2.4 循环进阶控制：loop_control

通过`loop_control`参数实现循环行为的精细化控制，解决复杂场景的循环需求：

|参数|核心作用|示例|
|:--|:--|:--|
|`label`|自定义循环执行时的日志输出标签，简化冗长的字典循环输出|`loop_control: { label: "{{ item.name }}" }`|
|`index_var`|自定义循环索引变量名，替代默认的`ansible_loop.index`|`loop_control: { index_var: "loop_idx" }`|
|`pause`|每次循环迭代之间的暂停时间，单位秒|`loop_control: { pause: 2 }`|
|`loop_var`|自定义迭代元素变量名，替代默认的`item`，解决嵌套循环的变量名冲突|`loop_control: { loop_var: "user_item" }`|

#### 标准示例：优化循环输出

```yaml
- name: 优化循环日志输出
  debug:
    msg: "创建用户：{{ item.name }}，UID：{{ item.uid }}"
  loop:
    - name: ops
      uid: 1001
      shell: /bin/bash
    - name: monitor
      uid: 1002
      shell: /sbin/nologin
  loop_control:
    label: "{{ item.name }}"  # 仅输出用户名，避免冗长的完整字典输出
```

### 6.2.5 最佳实践与避坑指南

1. **优先使用标准 loop**：除特殊场景外，生产环境优先使用官方推荐的 loop 关键字，替代传统 with_* 系列循环，保证语法一致性；
2. **避免循环嵌套**：尽量通过数据结构优化减少嵌套循环，嵌套循环会大幅提升剧本复杂度与调试成本；
3. **幂等性保障**：循环内的任务必须保持幂等性，避免多次循环执行产生副作用；
4. **输出优化**：字典循环必须配合`loop_control.label`简化输出，避免日志中出现大量冗余的完整字典内容；
5. **循环注册变量**：循环任务的 register 变量会包含`results`列表，存储每次迭代的执行结果，可通过`results`字段遍历循环执行结果。

---
