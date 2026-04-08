## 本章学习目标

本章将前序八章的所有理论知识、语法规范、工程化能力完整落地到企业级高频 Ansible 实战场景，严格遵循大纲指定的五大核心场景：**操作系统基线初始化、LAMP/LEMP 架构一键部署、安全加固与补丁批量下发、数据灾备自动化、公有云资源编排**。每个场景均遵循 Ansible 工程化最佳实践，严格保障幂等性、安全性、可复用性、可维护性，所有代码均可直接适配生产环境，无核心知识点遗漏，无冗余注水内容，实现从理论知识到生产落地的完整闭环。

---

## 9.1 操作系统基线初始化

### 9.1.1 场景核心目标

操作系统基线初始化是企业服务器上线的第一道标准化流程，核心目标是通过 Ansible 实现**新服务器的批量、标准化、合规化初始化**，消除服务器环境的异构性，满足企业安全合规要求，为后续业务部署提供统一、稳定、安全的基础运行环境。该场景是企业级 Ansible 落地的最基础、最高频场景，覆盖了前序章节的模块、变量、条件判断、Handlers、幂等性等所有核心知识点。

### 9.1.2 基线核心覆盖范围

本场景实现的标准化基线，覆盖企业级生产环境的核心合规要求：

1. 软件源标准化配置（Yum/Apt 源）
2. 系统时区与时间同步配置
3. SElinux 与防火墙基础合规配置
4. 系统内核参数优化（网络、文件、内存）
5. 系统用户与权限合规管理
6. SSH 服务安全加固
7. 系统基础依赖包统一安装
8. 不必要服务禁用与资源清理
9. 系统日志与审计规则配置

### 9.1.3 工程化实现方案

采用**Roles 角色化**实现，将基线拆分为独立的功能子角色，实现可插拔、可复用的模块化设计，目录结构如下：

```bash
ansible_os_baseline/
├── inventory/
│   ├── hosts
│   ├── group_vars/
│   │   └── all.yml
│   └── host_vars/
├── roles/
│   ├── repo_manage       # 软件源管理角色
│   ├── time_sync         # 时间同步角色
│   ├── security_base     # 安全基础配置角色
│   ├── kernel_optimize   # 内核参数优化角色
│   ├── user_manage       # 用户与权限管理角色
│   ├── ssh_hardening     # SSH安全加固角色
│   └── base_packages     # 基础依赖包安装角色
├── os_baseline.yml       # 主Playbook入口
└── ansible.cfg
```

### 9.1.4 核心代码实现

#### 1. 主 Playbook 入口：os_baseline.yml

```yaml
---
- name: 企业级操作系统标准化基线初始化
  hosts: all
  become: yes
  gather_facts: yes
  # 全局变量，可通过group_vars覆盖
  vars:
    baseline_env: "production"
    enable_ntp_sync: yes
    enable_ssh_hardening: yes
    enable_kernel_optimize: yes
  roles:
    - role: repo_manage
    - role: time_sync
      when: enable_ntp_sync | bool
    - role: base_packages
    - role: user_manage
    - role: security_base
    - role: kernel_optimize
      when: enable_kernel_optimize | bool
    - role: ssh_hardening
      when: enable_ssh_hardening | bool
```

#### 2. 核心角色示例：security_base（安全基础配置）

`roles/security_base/tasks/main.yml`

```yaml
---
- name: 配置SELinux状态
  selinux:
    state: "{{ selinux_state | default('permissive') }}"
    policy: targeted
  when: ansible_os_family == "RedHat"

- name: 停止并禁用不必要的系统服务
  service:
    name: "{{ item }}"
    state: stopped
    enabled: no
  loop: "{{ disable_services_list }}"
  ignore_errors: yes

- name: 配置防火墙默认策略
  firewalld:
    zone: public
    state: enabled
    permanent: yes
    immediate: yes
  when: ansible_os_family == "RedHat" and enable_firewalld | bool

- name: 放行SSH端口防火墙规则
  firewalld:
    port: "{{ ssh_port | default(22) }}/tcp"
    permanent: yes
    state: enabled
    immediate: yes
  when: ansible_os_family == "RedHat" and enable_firewalld | bool

- name: 配置系统日志轮转规则
  template:
    src: logrotate.conf.j2
    dest: /etc/logrotate.d/syslog
    mode: 0644
```

#### 3. 核心角色示例：kernel_optimize（内核参数优化）

`roles/kernel_optimize/tasks/main.yml`

```yaml
---
- name: 创建内核参数专属配置文件
  template:
    src: 99-os-baseline.conf.j2
    dest: /etc/sysctl.d/99-os-baseline.conf
    mode: 0644
    owner: root
    group: root
  notify: 重载内核参数

- name: 配置系统资源限制
  template:
    src: limits.conf.j2
    dest: /etc/security/limits.d/99-os-baseline.conf
    mode: 0644
```

`roles/kernel_optimize/handlers/main.yml`

```yaml
---
- name: 重载内核参数
  command: sysctl -p /etc/sysctl.d/99-os-baseline.conf
  changed_when: yes
```

### 9.1.5 最佳实践与避坑指南

1. **幂等性保障**：所有配置均采用声明式模块实现，禁止使用`echo >>`等增量写入命令，避免重复执行产生副作用；
2. **SSH 加固防锁定**：SSH 端口修改、禁用密码认证等高危操作，必须先验证密钥登录正常，配合`--check`干跑验证，避免批量锁定服务器；
3. **跨发行版适配**：通过`ansible_os_family`事实变量，实现 RedHat/Debian 系列的自动适配，一套剧本兼容多发行版；
4. **可插拔设计**：通过`when`条件实现功能模块的开关，不同环境可灵活选择启用 / 禁用特定基线项；
5. **前置测试**：新基线必须先在单台测试服务器验证，确认无锁定、无异常后，再批量执行；
6. **合规对齐**：基线配置需对齐企业等保合规、内部安全规范，所有配置项需可追溯、可审计。

---
