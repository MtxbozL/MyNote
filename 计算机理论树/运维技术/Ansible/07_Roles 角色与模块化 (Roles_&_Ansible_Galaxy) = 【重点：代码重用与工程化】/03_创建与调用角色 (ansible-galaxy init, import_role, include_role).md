
## 7.3 角色的创建与调用

### 7.3.1 标准化角色创建：ansible-galaxy init

Ansible 官方提供`ansible-galaxy init`命令，用于快速生成符合规范的角色目录结构，是创建角色的标准方式，避免手动创建目录的格式错误。

#### 基础语法

```c
ansible-galaxy init <角色名>
```

#### 核心参数与示例

```bash
# 1. 在当前目录生成标准nginx角色结构
ansible-galaxy init nginx

# 2. 指定角色生成路径
ansible-galaxy init nginx --init-path ./roles/

# 3. 生成最小化角色结构，跳过测试、README等非核心目录
ansible-galaxy init nginx --offline --init-path ./roles/

# 4. 查看init命令帮助
ansible-galaxy init --help
```

执行命令后，会自动生成 7.2 节定义的完整标准化目录结构与默认模板文件，无需手动创建。

### 7.3.2 Ansible 角色查找路径规则

调用角色前，必须明确 Ansible 的角色查找路径，否则会出现「角色未找到」的报错。Ansible 按**从高到低**的优先级顺序，在以下路径中查找角色：

1. 当前 Playbook 文件所在目录的`roles/`子目录（最常用，项目专属角色）；
2. 当前用户家目录的`~/.ansible/roles/`（用户级全局角色）；
3. 系统级目录`/usr/share/ansible/roles/`；
4. 系统级目录`/etc/ansible/roles/`；
5. `ANSIBLE_ROLES_PATH`环境变量指定的自定义路径。

**最佳实践**：项目专属角色统一存放在 Playbook 同级的`roles/`目录下，全局通用角色存放在`~/.ansible/roles/`目录，保持项目结构清晰。

### 7.3.3 角色的三种标准调用方式

Ansible 提供三种角色调用方式，适配不同场景，分别为 Play 级角色调用、静态导入`import_role`、动态包含`include_role`，覆盖大纲指定的全部调用方式。

#### 方式 1：Play 级角色调用（最常用、最基础）

在 Play 的`roles`字段中定义要调用的角色列表，是 Ansible 最经典的角色调用方式，适用于完整执行角色全量逻辑的场景。

##### 基础语法与示例

```yaml
---
- name: Play级调用nginx角色
  hosts: webservers
  become: yes
  # 角色调用列表
  roles:
    # 最简调用：直接写角色名
    - nginx
```

##### 进阶用法：调用时传递变量覆盖默认值

调用角色时，可通过`vars`字段传递变量，覆盖角色的 defaults 默认变量，实现角色的差异化配置：

```yaml
---
- name: 调用角色并传递自定义变量
  hosts: webservers
  become: yes
  roles:
    - role: nginx
      # 传递变量，覆盖角色默认值
      vars:
        nginx_listen_port: 8080
        nginx_worker_connections: 2048
    - role: php-fpm
      vars:
        php_version: 8.2
```

##### 核心执行规则

1. Play 级调用的角色，其任务会在 Play 的`tasks`字段之前执行；
2. 角色的`defaults`、`vars`变量会自动加载到 Play 的变量作用域中；
3. 角色的`handlers`会自动加载，Play 内的任务可直接 notify 调用；
4. 同一个角色多次调用，默认仅执行一次，如需多次执行，需设置`allow_duplicates: yes`在角色的`meta/main.yml`中。

#### 方式 2：静态导入角色：import_role

通过`import_role`模块，在 Play 的 tasks 任务列表中，静态导入角色，属于 Ansible 的静态加载机制，适用于需要精准控制角色执行时机、按条件执行角色的场景。

##### 基础语法与示例

```yaml
---
- name: import_role静态导入角色
  hosts: webservers
  become: yes
  tasks:
    - name: 执行系统前置初始化任务
      yum:
        name: ['epel-release', 'gcc', 'make']
        state: present

    # 静态导入nginx角色，在初始化任务完成后执行
    - name: 导入并执行nginx角色
      import_role:
        name: nginx
      # 传递变量覆盖默认值
      vars:
        nginx_listen_port: 8080

    - name: 角色执行完成后的后续任务
      uri:
        url: http://127.0.0.1:8080
        status_code: 200
```

##### 核心特性

1. **静态加载**：在 Playbook 解析阶段（执行前）就完成角色的导入与解析，与 Play 的原生内容完全融合；
2. **执行时机**：完全按任务列表的顺序执行，可精准控制角色在任意任务之间执行；
3. **条件与标签继承**：给`import_role`任务设置的`when`条件、`tags`标签，会自动继承给角色内的所有任务；
4. **语法校验**：解析阶段就会校验角色内的语法错误，提前发现问题，执行时不会出现语法报错。

#### 方式 3：动态包含角色：include_role

通过`include_role`模块，在 Play 的 tasks 任务列表中，动态包含角色，属于 Ansible 的动态加载机制，适用于需要基于运行时结果、循环、复杂条件动态加载角色的场景。

##### 基础语法与示例

```yaml
---
- name: include_role动态包含角色
  hosts: webservers
  become: yes
  tasks:
    - name: 检查Nginx是否已安装
      command: which nginx
      register: nginx_check
      ignore_errors: yes
      changed_when: no

    # 仅当Nginx未安装时，动态包含并执行nginx角色
    - name: 动态包含nginx角色
      include_role:
        name: nginx
      when: nginx_check.rc != 0
      # 传递变量
      vars:
        nginx_listen_port: 80
```

##### 循环批量包含角色示例

`include_role`支持循环，是其核心优势，`import_role`不支持循环：

```yaml
---
- name: 循环批量包含多个角色
  hosts: all
  become: yes
  vars:
    # 待安装的基础服务角色列表
    base_services:
      - ntp
      - firewalld
      - node_exporter
  tasks:
    - name: 循环安装基础服务
      include_role:
        name: "{{ item }}"
      loop: "{{ base_services }}"
```

---
