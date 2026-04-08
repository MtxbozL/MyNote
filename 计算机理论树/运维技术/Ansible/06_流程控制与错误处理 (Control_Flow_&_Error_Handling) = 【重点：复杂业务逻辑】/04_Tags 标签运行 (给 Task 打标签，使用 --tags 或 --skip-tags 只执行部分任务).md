### 6.4.1 严格定义

Tags 是 Ansible 提供的**任务精细化筛选执行机制**，通过给 Play、Task、Block、Role 打标签，执行剧本时通过`--tags`或`--skip-tags`参数，仅执行或跳过匹配标签的任务，无需修改剧本代码，即可实现同一套剧本的多场景差异化执行，适配增量更新、局部配置、故障修复等高频运维场景。

### 6.4.2 标签定义方式

标签支持多级别定义，遵循「继承规则」：上级定义的标签会自动继承给下级所有任务，同时支持单个对象绑定多个标签。

#### 1. Task 级标签（最常用）

给单个 Task 绑定标签，实现单任务的筛选执行：

```yaml
tasks:
  - name: 安装Nginx软件包
    yum:
      name: nginx
      state: present
    tags:
      - install
      - nginx
      - never

  - name: 分发Nginx配置文件
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    tags:
      - config
      - nginx

  - name: 重启Nginx服务
    service:
      name: nginx
      state: restarted
    tags:
      - restart
      - service
      - nginx
```

#### 2. Play 级标签

给整个 Play 绑定标签，Play 内所有 Task 会自动继承该标签：

```yaml
---
- name: Nginx全流程部署
  hosts: webservers
  become: yes
  tags:
    - nginx
    - web
  tasks:
    # 所有任务自动继承nginx、web标签
    - name: 安装Nginx
      yum:
        name: nginx
        state: present
```

#### 3. Block 级标签

给 Block 块绑定标签，块内所有 Task 自动继承该标签，实现批量标签管理：

```yaml
tasks:
  - name: Nginx部署块
    tags:
      - nginx
      - web
    block:
      - name: 安装Nginx
        yum:
          name: nginx
          state: present
      - name: 分发配置
        template:
          src: nginx.conf.j2
          dest: /etc/nginx/nginx.conf
```

### 6.4.3 核心执行参数与特殊标签

#### 1. 核心执行参数

|参数|核心作用|执行示例|
|:--|:--|:--|
|`--tags`|仅执行匹配指定标签的任务，多个标签用逗号分隔|`ansible-playbook nginx.yml --tags "config,restart"`|
|`--skip-tags`|跳过匹配指定标签的任务，执行其余所有任务|`ansible-playbook nginx.yml --skip-tags "install,never"`|
|`--list-tags`|列出剧本中所有定义的标签，不执行任何任务|`ansible-playbook nginx.yml --list-tags`|
|`--list-tasks --tags <标签>`|列出指定标签匹配的所有任务，用于范围校验|`ansible-playbook nginx.yml --list-tasks --tags "config"`|

#### 2. 内置特殊标签

Ansible 提供 5 个内置特殊标签，实现固定的执行逻辑，优先级高于自定义标签：

|特殊标签|核心作用|
|:--|:--|
|`always`|绑定该标签的任务，无论如何筛选都会执行，除非通过`--skip-tags always`强制跳过|
|`never`|绑定该标签的任务，仅当显式指定`--tags`匹配该标签时才会执行，默认永远不执行|
|`tagged`|匹配所有绑定了任意自定义标签的任务，排除无标签任务|
|`untagged`|仅匹配没有绑定任何标签的任务|
|`all`|匹配所有任务，默认执行逻辑，无需显式指定|

### 6.4.4 标准执行场景示例

基于前文的 Nginx 剧本标签定义，核心执行场景如下：

```bash
# 1. 仅执行配置更新相关任务
ansible-playbook nginx_deploy.yml --tags "config"

# 2. 仅执行Nginx重启任务
ansible-playbook nginx_deploy.yml --tags "restart"

# 3. 执行所有Nginx相关任务，跳过安装步骤
ansible-playbook nginx_deploy.yml --tags "nginx" --skip-tags "install"

# 4. 执行默认不执行的安装任务（绑定了never标签）
ansible-playbook nginx_deploy.yml --tags "install"

# 5. 执行所有任务，跳过重启步骤
ansible-playbook nginx_deploy.yml --skip-tags "restart"
```

### 6.4.5 最佳实践与命名规范

1. **标签分层设计**：按「业务线 - 模块 - 操作」三层结构设计标签，如`web-nginx-install`、`web-nginx-config`，避免标签混乱；
2. **最小粒度原则**：核心操作（安装、配置、重启、校验、初始化）必须单独绑定标签，实现精细化控制；
3. **never 标签使用**：一次性、高危操作（如初始化、数据清理、软件卸载）必须绑定`never`标签，避免误执行；
4. **always 标签慎用**：仅用于前置校验、环境检查等必须执行的任务，禁止滥用；
5. **禁止过度标签化**：避免给每个 Task 都绑定唯一标签，导致标签体系臃肿，按功能维度聚合标签。

---
