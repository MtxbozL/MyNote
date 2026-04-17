
>本节严格覆盖大纲指定的 4 种核心变量定义方式，同时补充配套的标准化用法，每种方式均明确作用域、语法规则、适用场景、标准示例与最佳实践。

### 5.2.1 Play 内 vars 块定义

**作用域**：当前 Play 内所有 Task 全局生效，优先级高于主机 / 组级变量，低于 Task 级变量与 Extra vars。

**语法规则**：在 Play 内通过`vars`字段定义，YAML 字典格式，支持字符串、数字、布尔值、列表、字典等全数据类型。

**标准示例**：

```yaml
---
- name: Play级vars块变量示例
  hosts: webservers
  remote_user: ops
  become: yes
  # Play级变量定义
  vars:
    nginx_version: 1.24.0
    nginx_root: /usr/share/nginx/html
    nginx_port: 80
    # 列表类型变量
    nginx_modules:
      - ngx_http_ssl_module
      - ngx_http_gzip_module
    # 字典类型变量
    nginx_config:
      worker_processes: auto
      worker_connections: 1024
      keepalive_timeout: 65

  tasks:
    - name: 安装指定版本Nginx
      yum:
        name: "nginx-{{ nginx_version }}"
        state: present

    - name: 渲染Nginx主配置
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
```

**最佳实践**：Play 内全局通用的参数、版本号、路径配置，优先通过 vars 块统一定义，避免 Task 内硬编码，提升剧本可维护性。

### 5.2.2 group_vars 与 host_vars 目录定义

**作用域**：Inventory 级全局生效，`group_vars`对指定主机组生效，`host_vars`对指定单个主机生效，主机级变量优先级高于组级变量。

**核心特性**：Ansible 会自动扫描与 Inventory 文件 / 目录同级的`group_vars`和`host_vars`目录，自动加载目录内的变量文件，无需手动导入，是企业级多环境、多分组变量管理的标准方案。

#### 标准化目录结构

```bash
inventory/
├── hosts                # 静态Inventory主文件
├── group_vars/          # 组变量目录
│   ├── all.yml          # all组全局变量，对所有主机生效
│   ├── webservers.yml   # webservers组专属变量
│   └── dbservers.yml    # dbservers组专属变量
└── host_vars/           # 主机变量目录
    ├── 192.168.1.101.yml  # 单个主机专属变量
    └── web02.example.com.yml
```

#### 变量文件示例

`group_vars/webservers.yml`：

```yaml
---
# webservers组全局变量
nginx_version: 1.24.0
nginx_root: /data/nginx/html
firewall_allowed_services:
  - http
  - https
```

`host_vars/192.168.1.101.yml`：

```yaml
---
# 单主机专属变量，覆盖组级同名变量
nginx_worker_processes: 8
nginx_worker_connections: 2048
```

**最佳实践**：

1. 公共变量写入`group_vars/all.yml`，按业务线、环境、功能维度拆分不同的组变量文件；
2. 仅单主机专属的差异化配置写入`host_vars`，避免过度使用主机变量导致维护成本上升；
3. 变量文件统一使用`.yml`后缀，YAML 格式编写，可读性与可扩展性优于 INI 格式。

### 5.2.3 命令行传入变量（-e/--extra-vars）

**作用域**：全局作用域，优先级最高，永远不会被其他变量覆盖，对整个执行过程的所有 Play、Task 均生效。

**语法规则**：通过`ansible-playbook`命令的`-e`或`--extra-vars`参数传入，支持 3 种传入格式：

1. 键值对格式：`key=value`，适用于简单变量；
2. JSON 格式：`'{"key": "value", "list": [1,2,3]}'`，适用于复杂数据类型；
3. YAML 文件格式：`@vars.yml`，从外部 YAML 文件批量加载变量。

**标准示例**：

```bash
# 1. 键值对格式传入简单变量
ansible-playbook nginx_deploy.yml -e "nginx_version=1.25.0 nginx_port=8080"

# 2. JSON格式传入复杂变量
ansible-playbook nginx_deploy.yml -e '{"nginx_version": "1.25.0", "nginx_modules": ["ssl", "gzip"]}'

# 3. 从外部YAML文件加载变量
ansible-playbook nginx_deploy.yml -e @prod_vars.yml
```

**适用场景**：

- 执行时动态指定敏感信息（如密码、密钥），避免写入剧本文件；
- 同一套剧本适配不同环境，通过命令行传入环境变量切换配置；
- 流水线 / CI/CD 场景中，动态传递构建号、版本号、部署路径等参数。

### 5.2.4 其他补充变量定义方式

#### 1. vars_files 外部变量文件导入

**作用域**：当前 Play 内生效，优先级与 Play 级 vars 块一致，适用于变量数量多、需拆分管理的场景。

**示例**：

```yaml
---
- name: vars_files导入外部变量
  hosts: webservers
  vars_files:
    - ./vars/common.yml
    - ./vars/nginx.yml
  tasks:
    - name: 安装Nginx
      yum:
        name: "nginx-{{ nginx_version }}"
        state: present
```

#### 2. set_fact 动态设置变量

**作用域**：当前 Play 内生效，优先级高于 Task 级变量，仅次于 Extra vars，用于在任务执行过程中动态生成、修改变量。

**示例**：

```yaml
- name: 动态设置变量
  set_fact:
    nginx_installed: yes
    current_time: "{{ ansible_date_time.iso8601 }}"
```

---
