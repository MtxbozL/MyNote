### 9.2.1 场景核心目标

LAMP（Linux + Apache + MySQL/MariaDB + PHP）与 LEMP（Linux + Nginx + MySQL/MariaDB + PHP）是企业 Web 服务的经典架构，本场景通过 Ansible Roles 实现架构的**一键式、标准化、可复用部署**，解决手动部署的步骤繁琐、环境异构、配置不统一、运维成本高的问题。场景深度融合 Roles、Jinja2 模板、Handlers、变量、条件判断等核心能力，是 Ansible 业务部署的经典落地场景。

### 9.2.2 工程化实现方案

采用**单组件单 Role**的角色化设计，实现架构组件的解耦与灵活组合，通过变量切换实现 LAMP 与 LEMP 架构的灵活切换，无需修改核心代码。目录结构如下：

```bash
ansible_web_stack/
├── inventory/
│   ├── hosts
│   └── group_vars/
│       └── all.yml
├── roles/
│   ├── nginx         # Nginx服务角色（LEMP）
│   ├── apache        # Apache服务角色（LAMP）
│   ├── mysql         # MySQL/MariaDB数据库角色
│   ├── php           # PHP-FPM服务角色
│   └── web_app       # Web应用部署角色
├── deploy_lemp.yml   # LEMP架构部署Playbook
├── deploy_lamp.yml   # LAMP架构部署Playbook
└── ansible.cfg
```

### 9.2.3 核心代码实现

#### 1. LEMP 架构主 Playbook：deploy_lemp.yml

```yaml
---
- name: 企业级LEMP架构一键部署
  hosts: web_servers
  become: yes
  gather_facts: yes
  vars_files:
    - ./vars/lemp_defaults.yml
  pre_tasks:
    - name: 安装系统依赖包
      yum:
        name: ['epel-release', 'gcc', 'make', 'openssl-devel']
        state: present
  roles:
    - role: nginx
    - role: mysql
    - role: php
    - role: web_app
  post_tasks:
    - name: 验证LEMP服务可用性
      uri:
        url: http://127.0.0.1/info.php
        status_code: 200
      register: health_check
      retries: 5
      delay: 3
      until: health_check is succeeded
    - name: 输出部署成功信息
      debug:
        msg: "LEMP架构部署成功，访问地址：http://{{ ansible_default_ipv4.address }}"
```

#### 2. 核心角色示例：mysql（数据库部署与安全初始化）

`roles/mysql/tasks/main.yml`

```yaml
---
- name: 安装MySQL服务与依赖
  yum:
    name: "{{ mysql_packages }}"
    state: present
  vars:
    mysql_packages:
      - mariadb-server
      - mariadb
      - python3-PyMySQL
      - mysql-connector-python

- name: 启动MySQL服务并设置开机自启
  service:
    name: mariadb
    state: started
    enabled: yes

- name: 检查MySQL root用户密码是否已设置
  stat:
    path: /root/.my.cnf
  register: mysql_root_config

- name: MySQL安全初始化 - 设置root密码
  mysql_user:
    name: root
    host: localhost
    password: "{{ mysql_root_password }}"
    login_unix_socket: /var/lib/mysql/mysql.sock
    state: present
  when: not mysql_root_config.stat.exists
  no_log: yes

- name: MySQL安全初始化 - 删除匿名用户
  mysql_user:
    name: ""
    host_all: yes
    state: absent
    login_user: root
    login_password: "{{ mysql_root_password }}"
  when: not mysql_root_config.stat.exists

- name: MySQL安全初始化 - 禁用root远程登录
  mysql_user:
    name: root
    host: "%"
    state: absent
    login_user: root
    login_password: "{{ mysql_root_password }}"
  when: not mysql_root_config.stat.exists

- name: MySQL安全初始化 - 删除test测试库
  mysql_db:
    name: test
    state: absent
    login_user: root
    login_password: "{{ mysql_root_password }}"
  when: not mysql_root_config.stat.exists

- name: 生成root用户本地登录配置文件
  template:
    src: my.cnf.j2
    dest: /root/.my.cnf
    mode: 0600
    owner: root
    group: root
  when: not mysql_root_config.stat.exists
  no_log: yes

- name: 创建应用数据库
  mysql_db:
    name: "{{ app_db_name }}"
    state: present
    encoding: utf8mb4
    collation: utf8mb4_general_ci

- name: 创建应用数据库用户
  mysql_user:
    name: "{{ app_db_user }}"
    password: "{{ app_db_password }}"
    priv: "{{ app_db_name }}.*:ALL"
    host: "{{ app_db_access_host | default('127.0.0.1') }}"
    state: present
  no_log: yes
```

#### 3. 核心角色示例：nginx（Nginx 服务部署与配置）

`roles/nginx/tasks/main.yml`

```yaml
---
- name: 安装Nginx
  yum:
    name: nginx
    state: present

- name: 创建Nginx配置目录
  file:
    path: /etc/nginx/conf.d
    state: directory
    mode: 0755

- name: 渲染Nginx主配置文件
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    mode: 0644
  notify: 重载Nginx

- name: 渲染PHP站点配置文件
  template:
    src: default.conf.j2
    dest: /etc/nginx/conf.d/default.conf
    mode: 0644
  notify: 重载Nginx

- name: 启动Nginx服务并设置开机自启
  service:
    name: nginx
    state: started
    enabled: yes
```

`roles/nginx/handlers/main.yml`

```yaml
---
- name: 重载Nginx
  service:
    name: nginx
    state: reloaded

- name: 重启Nginx
  service:
    name: nginx
    state: restarted
```

### 9.2.4 最佳实践与避坑指南

1. **敏感信息加密**：MySQL root 密码、应用数据库密码等敏感信息，必须通过`ansible-vault`加密，禁止明文存储在剧本或变量文件中；
2. **幂等性保障**：数据库初始化操作仅在首次执行时运行，通过状态文件判断避免重复执行，防止密码被覆盖、数据被误删；
3. **健康检查机制**：部署完成后必须添加服务可用性校验，通过`uri`模块验证 Web 服务、PHP 解析、数据库连接正常，确保部署闭环；
4. **配置与代码分离**：所有可配置项均通过变量定义，禁止硬编码在角色代码中，实现一套剧本适配开发、测试、生产多环境；
5. **版本锁定**：生产环境必须锁定 Nginx、MySQL、PHP 的软件版本，禁止使用`state: latest`，避免非预期的版本升级导致业务异常；
6. **架构灵活扩展**：通过角色解耦，可灵活替换 Web 服务器、数据库版本，支持分布式部署（Web 与数据库分离），适配不同规模的业务场景。

---
