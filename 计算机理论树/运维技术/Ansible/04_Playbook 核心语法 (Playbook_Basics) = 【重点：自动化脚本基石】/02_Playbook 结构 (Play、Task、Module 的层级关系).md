### 4.2.1 严格定义

Playbook 是 Ansible 的自动化剧本文件，以`.yml`/`.yaml`为文件后缀，是**一个或多个 Play 的有序集合**。通过声明式语法定义完整的自动化流程，实现多主机、多任务、多步骤的批量自动化操作，支持版本控制、代码复用、团队协作，是企业级 Ansible 自动化的核心载体。

与 Ad-Hoc 命令的核心区别：Ad-Hoc 仅能实现单任务、一次性操作，无持久化能力；Playbook 可实现多任务有序编排、逻辑控制、变量复用、版本管理，是复杂自动化场景的唯一标准方案。

### 4.2.2 核心层级关系

Playbook 采用严格的四层层级结构，从顶层到底层依次为：

```bash
Playbook文件（顶层容器）
└── Play 1（定义目标范围与全局配置）
    └── Task 1（最小执行单元，调用单个模块）
        └── Module（底层执行载体，定义目标状态）
    └── Task 2
        └── Module
└── Play 2（针对另一组主机的独立操作流程）
    └── Task列表
```

1. **Playbook 文件**：顶层容器，是一个 YAML 列表，每个列表元素对应一个独立的 Play，一个 Playbook 可包含多个 Play，实现跨主机组的多流程编排；
2. **Play**：自动化流程的逻辑单元，核心定义「在哪些主机上、以什么身份、执行哪些任务」，明确操作的目标范围与全局配置；
3. **Task**：Play 中的最小执行单元，为有序执行的任务列表，每个 Task 对应一个具体的操作，仅能调用一个 Ansible 模块，默认按从上到下的顺序串行执行；
4. **Module**：Task 的底层执行载体，即第三章覆盖的所有 Ansible 功能模块，每个 Task 通过模块参数定义目标系统状态，实现具体操作。

### 4.2.3 Play 的核心组成字段

|字段类型|字段名|核心作用|必选 / 可选|
|:--|:--|:--|:--|
|必选字段|`hosts`|定义 Play 的目标主机范围，匹配规则与 Ad-Hoc 的 host-pattern 完全一致，支持`all`、分组名、IP、通配符、正则等|必选|
|高频可选字段|`name`|Play 的名称，用于执行时的日志输出，提升可读性与可维护性，**生产环境强制要求**|可选|
||`remote_user`|远程连接被控节点的系统用户名，对应第二章的`ansible_user`参数|可选|
||`become`|是否开启权限提升，布尔值，对应第一章的提权机制|可选|
||`become_method`|提权方式，默认`sudo`，可选`su`、`doas`等|可选|
||`become_user`|提权至的目标用户，默认`root`|可选|
||`gather_facts`|是否采集被控节点的 Facts 系统信息，布尔值，默认`yes`|可选|
||`vars`|Play 级别的变量定义|可选|
||`tasks`|Play 的核心任务列表，有序执行|可选|
||`handlers`|触发器任务列表|可选|
||`tags`|Play 级别的标签，用于任务筛选执行|可选|

### 4.2.4 Playbook 最小化合法结构

```yaml
---
# 顶层为列表，每个元素为一个Play
- name: 最小化Play示例
  hosts: webservers
  remote_user: ops
  become: yes
  tasks:
    - name: 安装Nginx
      yum:
        name: nginx
        state: present
```

> 说明：`---`为 YAML 文档的开始标识，是 Ansible Playbook 的规范写法，可选但推荐添加。

---
