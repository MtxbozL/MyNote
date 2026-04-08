## 本章学习目标

本章完整覆盖 Ansible 复杂业务逻辑编排的全体系核心能力，严格遵循大纲顺序，从条件判断、循环机制，到**必考核心特性 Handlers 触发器**、标签精细化执行控制、全场景错误处理机制，最终完成暂停与交互控制的实战落地，无核心知识点遗漏。本章建立生产级高可用、高容错、可灵活调度的自动化剧本编写能力，是从基础静态剧本到企业级复杂业务流程编排的核心进阶环节，承接前序变量、Playbook 基础语法，为后续角色化、高级特性与实战场景提供核心逻辑支撑。

---

## 6.1 条件判断（Conditionals）

### 6.1.1 严格定义

条件判断是 Ansible 实现分支逻辑控制的核心机制，通过`when`语句定义任务执行的前置布尔条件，**仅当条件表达式计算结果为真时，才执行对应任务，否则直接跳过**。基于该机制，可实现被控节点差异化适配、执行结果分支流转、异常场景逻辑控制，是动态剧本的核心基础能力。

### 6.1.2 核心语法规则

1. `when`语句与 Task 同级定义，无需包裹`{{ }}`分隔符，Ansible 会自动将`when`后的内容解析为 Jinja2 表达式，这是区别于普通变量引用的核心规则，也是高频踩坑点；
2. 支持 Jinja2 全量比较运算符、逻辑运算符、内置测试函数；
3. 可直接引用 Ansible 变量、Facts、Register 注册变量、魔法变量，无需额外格式包裹。

#### 核心运算符与测试函数

| 类型     | 核心可用项                                              | 说明                    |
| :----- | :------------------------------------------------- | :-------------------- |
| 比较运算符  | `==`、`!=`、`>`、`<`、`>=`、`<=`                        | 数值与字符串的等值、大小比较        |
| 逻辑运算符  | `and`、`or`、`not`                                   | 多条件组合与取反              |
| 归属判断   | `in`、`not in`                                      | 判断元素是否在列表 / 字典 / 字符串中 |
| 定义性测试  | `defined`、`undefined`                              | 判断变量是否已定义，避免未定义变量报错   |
| 执行状态测试 | `succeeded`/`success`、`failed`、`changed`、`skipped` | 判断任务执行结果状态            |
| 路径测试   | `exists`、`isdir`、`isfile`、`islink`                 | 判断被控节点路径的类型与存在性       |

### 6.1.3 全场景实战示例

#### 示例 1：基于 Facts 的跨环境适配

最常用的条件判断场景，基于被控节点的系统特性执行差异化任务，完美适配多发行版、多架构环境：

```yaml
---
- name: 基于Facts的跨发行版软件安装
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
        update_cache: yes
      when: ansible_os_family == "Debian"

    - name: 仅x86_64架构安装Node Exporter
      yum:
        name: node_exporter
        state: present
      when: ansible_architecture == "x86_64"
```

#### 示例 2：基于 Register 注册变量的条件流转

联动第五章的 Register 机制，基于前序任务的执行结果、返回码、输出内容，决定后续任务是否执行，实现闭环流程控制：

```yaml
---
- name: 基于注册变量的配置变更流程
  hosts: webservers
  become: yes
  tasks:
    - name: 校验Nginx配置合法性
      command: nginx -t
      register: nginx_config_check
      ignore_errors: yes
      changed_when: no

    - name: 仅配置校验通过时，平滑重载Nginx
      service:
        name: nginx
        state: reloaded
      when: nginx_config_check is succeeded

    - name: 配置校验失败时，输出错误日志并中断执行
      fail:
        msg: "Nginx配置校验失败，错误信息：{{ nginx_config_check.stderr }}"
      when: nginx_config_check is failed
```

#### 示例 3：多条件组合与变量判断

实现复杂的多条件分支逻辑，同时处理变量未定义的边界场景：

```yaml
---
- name: 多条件组合判断示例
  hosts: webservers
  become: yes
  vars:
    nginx_ssl_enable: yes
    nginx_version: "1.24.0"
  tasks:
    - name: 安装SSL模块（开启SSL且版本大于1.20）
      yum:
        name: nginx-mod-ssl
        state: present
      when:
        - nginx_ssl_enable is defined
        - nginx_ssl_enable | bool
        - nginx_version is version('1.20.0', '>=')

    - name: 生产环境关闭SELinux
      selinux:
        state: permissive
        policy: targeted
      when:
        - env is defined
        - env == "production"
        - ansible_os_family == "RedHat"
```

### 6.1.4 进阶用法与最佳实践

1. **条件复用**：多个 Task 共用相同条件时，可将条件定义在 Block 块级，避免重复代码；
2. **版本比较**：使用`version`过滤器进行软件版本的精准比较，替代字符串等值判断；
3. **边界防护**：所有引用自定义变量的条件，必须先通过`defined`测试判断变量是否存在，避免剧本执行中断；
4. **布尔值处理**：通过`| bool`过滤器强制转换为布尔值，避免字符串 "yes"/"no" 导致的判断异常。

---
