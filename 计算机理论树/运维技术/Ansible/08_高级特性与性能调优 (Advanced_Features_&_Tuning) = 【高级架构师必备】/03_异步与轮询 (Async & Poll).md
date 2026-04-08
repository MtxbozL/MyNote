### 8.3.1 核心定义与设计价值

Ansible 默认采用**同步执行模型**：控制节点发起任务后，保持 SSH 连接等待被控节点任务执行完成，再返回结果。对于长耗时任务（如大数据备份、源码编译安装、大文件传输、数据库大表变更），会出现两个核心问题：

1. SSH 会话超时，导致任务中断，执行状态不可控；
2. 控制节点长期占用 SSH 连接，无法执行其他操作，并发能力下降。

**异步与轮询（Async & Poll）** 是 Ansible 针对长耗时任务的解决方案，核心原理是：将任务放到被控节点的后台异步执行，控制节点无需保持 SSH 长连接，定期轮询检查任务执行状态，直到任务完成或超时，彻底解决长耗时任务的稳定性问题。

### 8.3.2 核心语法与参数体系

异步任务通过两个核心参数控制：

|参数|核心作用|必选|默认值|
|:--|:--|:--|:--|
|`async`|任务的最大超时时间，单位秒，超过该时间任务会被强制终止，标识该任务为异步执行|是|无|
|`poll`|轮询间隔时间，单位秒，控制节点每隔该时间检查一次任务执行状态|否|10 秒|

#### 基础语法示例

```yaml
---
- name: 异步执行源码编译安装
  hosts: all
  become: yes
  tasks:
    - name: 异步编译安装Nginx源码
      shell: |
        wget https://nginx.org/download/nginx-1.24.0.tar.gz
        tar -zxf nginx-1.24.0.tar.gz
        cd nginx-1.24.0
        ./configure --prefix=/usr/local/nginx --with-http_ssl_module
        make -j {{ ansible_processor_vcpus }}
        make install
      async: 1800  # 最大超时30分钟
      poll: 30      # 每30秒轮询一次状态
      register: nginx_compile_async
```

### 8.3.3 三大异步执行模式与实战场景

#### 模式 1：异步 + 轮询（默认模式）

控制节点提交异步任务后，按`poll`设置的间隔定期轮询任务状态，阻塞等待任务完成后，再执行后续任务，适用于长耗时但后续任务依赖其执行结果的场景，如大纲指定的源码编译安装、大数据备份。

**标准示例：大数据备份异步执行**

```yaml
---
- name: 异步执行数据库全量备份
  hosts: dbservers
  become: yes
  tasks:
    - name: 异步执行MySQL全量备份
      shell: mysqldump -u root --all-databases --single-transaction | gzip > /data/backup/mysql_full_{{ ansible_date_time.date }}.sql.gz
      async: 3600  # 最大超时1小时
      poll: 60      # 每分钟检查一次状态
      register: mysql_backup_async

    - name: 输出备份执行结果
      debug:
        var: mysql_backup_async

    - name: 备份完成后拉取备份文件到控制节点
      fetch:
        src: /data/backup/mysql_full_{{ ansible_date_time.date }}.sql.gz
        dest: ./backup/
        flat: yes
```

#### 模式 2：异步 + 忽略结果（Fire and Forget）

设置`poll: 0`，控制节点提交异步任务后，立即进入后续任务，完全不等待任务执行结果，也不检查状态，适用于无需关注执行结果、无需后续依赖的后台长耗时任务，如日志清理、大文件同步、后台数据归档。

**标准示例**：

```yaml
---
- name: 异步执行后台日志清理任务
  hosts: all
  become: yes
  tasks:
    - name: 后台清理30天前的日志文件
      shell: find /var/log -type f -mtime +30 -delete
      async: 1800
      poll: 0  # 提交后立即继续，不等待结果

    - name: 立即执行后续任务，无需等待清理完成
      yum:
        name: ['logrotate']
        state: present
```

#### 模式 3：异步 + 手动检查状态

先提交异步任务，执行其他不相关的任务，后续通过`async_status`模块手动检查异步任务的执行状态，最大化利用执行时间，适用于多任务并行、非强依赖的场景。

**标准示例**：

```yaml
---
- name: 异步任务+手动状态检查
  hosts: all
  become: yes
  tasks:
    # 1. 提交长耗时异步编译任务，不等待
    - name: 提交Nginx编译异步任务
      shell: |
        cd /opt/src/nginx-1.24.0
        ./configure --prefix=/usr/local/nginx
        make && make install
      async: 1800
      poll: 0
      register: nginx_async_task

    # 2. 执行其他不相关的任务，利用编译时间
    - name: 安装Nginx依赖包
      yum:
        name: ['openssl-devel', 'pcre-devel', 'zlib-devel']
        state: present

    - name: 配置Nginx系统服务
      template:
        src: nginx.service.j2
        dest: /usr/lib/systemd/system/nginx.service
        mode: 0644

    # 3. 手动检查异步任务执行状态，等待完成
    - name: 检查Nginx编译任务状态
      async_status:
        jid: "{{ nginx_async_task.ansible_job_id }}"
      register: job_result
      retries: 60
      delay: 30
      until: job_result.finished

    # 4. 编译完成后执行后续依赖任务
    - name: 启动Nginx服务
      service:
        name: nginx
        state: started
        enabled: yes
```

### 8.3.4 核心原理与最佳实践

1. **异步任务执行原理**：
    
    - Ansible 在被控节点创建后台任务，生成唯一的`ansible_job_id`，并将任务状态写入临时文件；
    - 控制节点通过`ansible_job_id`，使用`async_status`模块查询任务执行状态；
    - 任务完成 / 超时后，自动清理临时状态文件。
    
2. **最佳实践**：
    
    - **超时时间合理设置**：`async`参数必须设置为大于任务最大预期执行时间，避免任务被提前终止；
    - **轮询间隔优化**：短耗时任务设置小间隔，长耗时任务设置大间隔，减少不必要的 SSH 请求；
    - **幂等性保障**：异步任务必须保证幂等性，避免任务中断重试导致的副作用；
    - **结果校验**：关键任务必须通过`async_status`检查最终执行结果，避免任务失败未被发现；
    - **避免 SSH 超时**：超过 10 分钟的长耗时任务，必须使用异步执行，禁止同步执行；
    - **并发控制**：大规模集群的异步任务，需配合`forks`参数控制并发数，避免被控节点负载过高。
    
3. **避坑指南**：
    
    - 异步任务的`become`提权配置必须与任务需求匹配，避免后台任务权限不足；
    - 异步任务中的相对路径需替换为绝对路径，避免后台执行时工作目录异常；
    - 被控节点重启后，异步任务会丢失，无法继续跟踪状态，重启相关任务禁止使用异步；
    - `poll: 0`模式下，任务失败不会导致 Playbook 执行失败，需自行实现结果校验与告警。
    

---
