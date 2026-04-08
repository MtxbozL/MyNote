
Roles 基于「约定大于配置」的核心设计，采用固定的标准化目录结构，Ansible 会自动识别并加载对应目录中的内容，无需手动 import。每个目录对应一个明确的功能维度，其中`tasks`目录为必填项，其余目录为可选，无对应内容时可省略。

### 7.2.1 标准目录树

以`nginx`角色为例，完整的标准化目录结构如下：

```bash
roles/
└── nginx/                    # 角色根目录，目录名即为角色名
    ├── tasks/                # 【必填】角色的核心任务列表
    │   └── main.yml          # 角色任务入口文件，Ansible自动加载
    ├── handlers/             # 角色的触发器任务列表
    │   └── main.yml          # 触发器入口文件，自动加载
    ├── templates/            # 角色的Jinja2模板文件
    ├── files/                # 角色的静态文件，用于copy/fetch模块
    ├── vars/                 # 角色的高优先级变量
    │   └── main.yml          # 角色变量定义文件，自动加载
    ├── defaults/             # 角色的低优先级默认变量
    │   └── main.yml          # 角色默认变量定义文件，自动加载
    ├── meta/                 # 角色的元数据与依赖定义
    │   └── main.yml          # 元数据入口文件，自动加载
    ├── tests/                # 角色的测试用例与测试剧本
    ├── README.md             # 角色的说明文档、使用方法、变量说明
    └── LICENSE               # 角色的开源协议声明
```

### 7.2.2 核心目录逐一定义与解析

#### 1. tasks 目录【必填】

**核心作用**：角色的核心执行逻辑入口，定义角色的所有任务列表，是角色的主体部分，Ansible 执行角色时，会自动加载并执行`tasks/main.yml`文件中的任务。

**标准示例**：`roles/nginx/tasks/main.yml`

```yaml
---
- name: 安装Nginx软件包
  yum:
    name: "{{ nginx_package_name }}"
    state: "{{ nginx_package_state }}"

- name: 创建Nginx配置目录
  file:
    path: "{{ nginx_config_dir }}"
    state: directory
    owner: root
    group: root
    mode: 0755

- name: 渲染Nginx主配置文件
  template:
    src: nginx.conf.j2
    dest: "{{ nginx_config_dir }}/nginx.conf
    mode: 0644
  notify: 重载Nginx服务

- name: 启动Nginx服务并设置开机自启
  service:
    name: "{{ nginx_service_name }}"
    state: "{{ nginx_service_state }}"
    enabled: "{{ nginx_service_enabled }}"
```

**补充规则**：支持任务拆分，可在 tasks 目录下创建多个 yml 文件，通过`import_tasks`/`include_tasks`在`main.yml`中引入，实现任务的模块化拆分，避免单文件过长。

示例：

```yaml
# tasks/main.yml
---
- import_tasks: install.yml
- import_tasks: config.yml
- import_tasks: service.yml
```

#### 2. handlers 目录

**核心作用**：定义角色专属的触发器任务，与第六章的 Handlers 机制完全一致，用于角色内任务的变更触发操作（如服务重启、重载），Ansible 会自动加载`handlers/main.yml`中的触发器，角色内的任务可直接通过`notify`调用。

**标准示例**：`roles/nginx/handlers/main.yml`

```yaml
---
- name: 重载Nginx服务
  service:
    name: nginx
    state: reloaded

- name: 重启Nginx服务
  service:
    name: nginx
    state: restarted
```

**核心特性**：角色内的 Handlers 仅在当前角色内生效，跨角色调用需通过完全限定名，避免命名冲突。

#### 3. templates 目录

**核心作用**：存放角色专属的 Jinja2 模板文件，角色内的`template`模块可直接引用模板文件名，无需写绝对路径，Ansible 会自动在该目录下查找对应模板文件。

**使用示例**：

- 模板文件存放路径：`roles/nginx/templates/nginx.conf.j2`
- 任务中直接引用：

    ```yaml
    - name: 渲染Nginx配置
      template:
        src: nginx.conf.j2  # 直接写文件名，无需路径
        dest: /etc/nginx/nginx.conf
    ```
    

#### 4. files 目录

**核心作用**：存放角色专属的静态文件，如二进制包、静态配置文件、证书文件等，角色内的`copy`模块可直接引用文件名，无需写绝对路径，Ansible 会自动在该目录下查找对应文件。

**使用示例**：

- 静态文件存放路径：`roles/nginx/files/index.html`
- 任务中直接引用：

    ```yaml
    - name: 复制网站首页
      copy:
        src: index.html  # 直接写文件名，无需路径
        dest: /usr/share/nginx/html/index.html
    ```
    

#### 5. vars 目录

**核心作用**：定义角色的高优先级内置变量，Ansible 会自动加载`vars/main.yml`中的变量，**优先级高于 Inventory 主机 / 组变量，仅低于 Task 级变量、Extra vars**，不可被用户的 Inventory 变量覆盖，用于角色内固定的、不希望用户修改的核心变量。

**标准示例**：`roles/nginx/vars/main.yml`

```yaml
---
# 角色固定变量，不希望用户修改
nginx_config_dir: /etc/nginx
nginx_service_name: nginx
nginx_user: nginx
nginx_group: nginx
```

**核心规则**：与第五章变量优先级完全呼应，vars 目录变量优先级远高于 defaults 目录，用于角色核心固定参数。

#### 6. defaults 目录

**核心作用**：定义角色的低优先级默认变量，Ansible 会自动加载`defaults/main.yml`中的变量，**优先级为全变量体系最低**，仅当用户没有通过任何更高优先级的方式定义同名变量时生效，用于给角色的参数提供默认值，允许用户通过 Inventory 变量、Play 变量、命令行参数轻松覆盖。

**标准示例**：`roles/nginx/defaults/main.yml`

```yaml
---
# 角色默认变量，用户可任意覆盖
nginx_package_name: nginx
nginx_package_state: present
nginx_service_state: started
nginx_service_enabled: yes
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_listen_port: 80
```

**最佳实践**：角色的所有可配置参数，都应在 defaults 目录中设置合理的默认值，让角色开箱即用，同时允许用户按需覆盖，提升角色的灵活性与兼容性。

#### 7. meta 目录

**核心作用**：定义角色的元数据信息，包括作者、版本、兼容操作系统、开源协议、**前置依赖**等，Ansible 会自动加载`meta/main.yml`中的内容，执行角色前会自动处理定义的依赖关系。

**标准基础示例**：`roles/nginx/meta/main.yml`

```yaml
---
galaxy_info:
  author: 作者名称
  description: 标准化Nginx部署角色
  company: 所属组织
  license: MIT
  min_ansible_version: 2.14
  # 兼容的操作系统平台
  platforms:
    - name: EL
      versions:
        - 8
        - 9
    - name: Debian
      versions:
        - 11
        - 12
    - name: Ubuntu
      versions:
        - 22.04
        - 24.04
  galaxy_tags:
    - nginx
    - web
    - http
# 角色依赖定义，7.6节详细讲解
dependencies: []
```

---
