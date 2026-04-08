### 8.2.1 核心定义与设计价值

任务委派是 Ansible 提供的跨节点执行调度能力，通过`delegate_to`关键字，将原本需要在被控节点上执行的任务，委派到指定的目标节点（控制节点、堡垒机、负载均衡器、API 网关、数据库节点等）执行，同时保留当前被控节点的上下文与变量，是实现跨节点协同、闭环业务流程编排的核心能力。

**核心解决的场景痛点**：

1. 应用发布闭环：更新应用前，先将节点从负载均衡器下线，更新完成后再重新上线（大纲指定的 F5 负载均衡场景）；
2. 本地 API 调用：在控制节点调用云厂商 API、CMDB 接口，无需在所有被控节点安装 API 依赖；
3. 跨节点数据同步：从数据库主节点拉取备份，分发到从节点；
4. 集中式证书签发：在控制节点统一签发 SSL 证书，再分发到对应 Web 节点；
5. 变更前置校验：在堡垒机 / 监控节点校验被控节点的端口、服务可用性。

### 8.2.2 核心语法与执行原理

#### 1. 标准委派语法：delegate_to

`delegate_to`与 Task 同级定义，指定任务委派的目标节点，目标节点必须在 Inventory 中已定义，Ansible 会自动使用该节点的连接参数建立会话。

**基础示例**：

```yaml
---
- name: 任务委派基础示例
  hosts: webservers
  tasks:
    # 该任务会委派到控制节点localhost执行，而非webservers节点
    - name: 在控制节点记录部署日志
      lineinfile:
        path: /opt/ansible/deploy_logs.log
        line: "{{ ansible_date_time.iso8601 }} - 开始部署节点 {{ inventory_hostname }}"
        create: yes
      delegate_to: localhost
```

#### 2. 本地操作简写：local_action

`local_action`是`delegate_to: localhost`的简写方式，专门用于将任务委派到控制节点本地执行，语法更简洁。

**等效示例**：

```yaml
# 方式1：delegate_to标准写法
- name: 本地执行API调用
  uri:
    url: https://api.example.com/notify
    method: POST
    body: {"node": "{{ inventory_hostname }}", "status": "deploying"}
    body_format: json
  delegate_to: localhost

# 方式2：local_action简写写法
- name: 本地执行API调用
  local_action:
    module: uri
    url: https://api.example.com/notify
    method: POST
    body: {"node": "{{ inventory_hostname }}", "status": "deploying"}
    body_format: json
```

#### 3. 核心执行原理

1. **变量上下文**：委派执行的任务，仍使用当前被控节点的变量（`inventory_hostname`、`ansible_default_ipv4`等），而非委派节点的变量，这是任务委派的核心设计；
2. **连接规则**：委派任务使用委派目标节点的 Inventory 连接参数（SSH 端口、用户、密钥等），委派到[localhost](https://localhost)时，默认使用 local 连接插件，无需 SSH 连接；
3. **执行范围**：默认情况下，每个被控节点的委派任务都会独立执行一次，如需仅执行一次，需配合`run_once: yes`参数。

### 8.2.3 核心实战场景：F5 负载均衡节点上下线

大纲指定的核心场景，完整实现 Web 节点发布的闭环流程：发布前将节点从 F5 负载均衡池下线，等待流量切走后执行应用更新，更新完成后验证服务可用性，再重新上线到负载均衡池。

```yaml
---
- name: Web节点滚动发布与F5负载均衡上下线
  hosts: webservers
  become: yes
  serial: 1  # 串行执行，每次仅更新1台节点，保障业务可用性
  tasks:
    - name: 将当前节点从F5负载均衡池下线
      local_action:
        module: bigip_pool_member
        provider:
          server: f5.example.com
          user: "{{ f5_admin_user }}"
          password: "{{ f5_admin_password }}"
          validate_certs: no
        pool: web_pool
        partition: Common
        name: "{{ inventory_hostname }}"
        address: "{{ ansible_default_ipv4.address }}"
        port: 80
        state: disabled
      delegate_to: localhost
      run_once: no

    - name: 等待F5连接耗尽，无新流量进入
      pause:
        seconds: 30

    - name: 停止应用服务
      service:
        name: nginx
        state: stopped

    - name: 更新应用代码包
      unarchive:
        src: ./packages/app_v2.1.0.tar.gz
        dest: /data/app/
        owner: nginx
        group: nginx

    - name: 启动应用服务
      service:
        name: nginx
        state: started
        enabled: yes

    - name: 本地验证应用服务可用性
      uri:
        url: http://127.0.0.1/health
        status_code: 200
      register: health_check
      retries: 5
      delay: 3
      until: health_check is succeeded

    - name: 将当前节点重新上线到F5负载均衡池
      local_action:
        module: bigip_pool_member
        provider:
          server: f5.example.com
          user: "{{ f5_admin_user }}"
          password: "{{ f5_admin_password }}"
          validate_certs: no
        pool: web_pool
        partition: Common
        name: "{{ inventory_hostname }}"
        address: "{{ ansible_default_ipv4.address }}"
        port: 80
        state: enabled
      delegate_to: localhost

    - name: 等待负载均衡健康检查通过，流量恢复
      pause:
        seconds: 15
```

### 8.2.4 进阶特性与最佳实践

#### 1. run_once 配合委派执行

通过`run_once: yes`参数，让委派任务仅在第一个被控节点上执行一次，而非所有节点重复执行，适用于全局一次性操作（如全局 API 通知、数据库批量更新）。

**示例**：

```yaml
- name: 发布前全局API通知
  uri:
    url: https://api.example.com/deploy/start
    method: POST
  delegate_to: localhost
  run_once: yes  # 仅执行一次，而非所有web节点都执行
```

#### 2. 委派任务的提权配置

委派任务默认继承 Play 的提权配置，如需单独配置，可在任务中单独设置`become`参数，适配委派节点的权限要求。

**示例**：

```yaml
- name: 在堡垒机执行特权命令
  command: /opt/scripts/health_check.sh {{ inventory_hostname }}
  delegate_to: jump_host.example.com
  become: yes
  become_user: root
```

#### 3. 最佳实践与避坑指南

1. **依赖前置校验**：委派任务所需的 Python 模块、命令工具，必须提前在委派节点安装，如 F5 模块需安装`f5-sdk`，API 调用需安装`python-requests`；
2. **连接可达性**：委派目标节点必须在 Inventory 中正确定义，控制节点与委派节点之间的网络、认证必须正常；
3. **避免过度委派**：仅跨节点协同、集中式操作使用委派，节点本地操作禁止使用委派，避免性能损耗；
4. **串行执行配合**：滚动发布、负载均衡上下线场景，必须配合`serial`参数串行执行，避免全量节点同时下线导致业务中断；
5. **敏感信息保护**：委派任务中的 API 密码、负载均衡管理员密码，必须通过 Ansible Vault 加密，禁止明文存储；
6. **本地连接优化**：频繁委派到[localhost](https://localhost)的场景，可在 Inventory 中为[localhost](https://localhost)设置`ansible_connection=local`，避免不必要的 SSH 连接开销。

---
