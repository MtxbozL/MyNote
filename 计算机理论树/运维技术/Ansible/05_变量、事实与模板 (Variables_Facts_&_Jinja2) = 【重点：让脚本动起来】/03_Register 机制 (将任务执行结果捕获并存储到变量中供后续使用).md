### 5.3.1 严格定义

Register 是 Ansible 提供的任务结果捕获机制，可将任意 Task 的执行结果（包括返回码、标准输出、标准错误、状态信息等）完整捕获并存储到自定义变量中，供后续 Task 的条件判断、参数传递、结果输出使用，是实现复杂业务逻辑、错误处理、动态流程控制的核心基础。

### 5.3.2 核心语法与标准结构

**基础语法**：在 Task 中通过`register`关键字指定变量名，即可捕获该 Task 的执行结果。

```yaml
- name: 执行系统命令
  command: uptime
  register: uptime_result  # 将执行结果捕获到uptime_result变量中
```

#### Register 变量的标准核心字段

所有 Register 变量均包含以下固定核心字段，是后续处理的核心依据：

|字段名|数据类型|核心含义|
|:--|:--|:--|
|`rc`|整数|命令 / 模块执行的返回码，0 = 执行成功，非 0 = 执行失败（ignore_errors 时仍可获取）|
|`stdout`|字符串|执行的标准输出内容，单行 / 多行文本|
|`stdout_lines`|列表|按换行符拆分的标准输出列表，逐行处理时使用|
|`stderr`|字符串|执行的标准错误内容|
|`stderr_lines`|列表|按换行符拆分的标准错误列表|
|`changed`|布尔值|任务是否标记为变更状态|
|`failed`|布尔值|任务是否标记为失败状态|
|`skipped`|布尔值|任务是否被跳过|
|`ansible_facts`|字典|模块返回的 Facts 信息，部分模块会自动注入|

### 5.3.3 全场景实战示例

#### 示例 1：结果捕获与调试输出

通过`debug`模块输出 Register 变量的内容，用于调试与结果查看：

```yaml
---
- name: Register捕获与调试示例
  hosts: all
  tasks:
    - name: 查看磁盘使用率
      command: df -h /
      register: disk_usage

    - name: 输出磁盘使用率结果
      debug:
        var: disk_usage  # 输出完整的Register变量结构

    - name: 仅输出标准输出内容
      debug:
        msg: "根分区使用率：{{ disk_usage.stdout }}"
```

#### 示例 2：结合条件判断实现流程控制

基于 Register 变量的返回码、输出内容，通过`when`条件判断后续任务是否执行，是第六章流程控制的核心前置基础：

```yaml
---
- name: Register结合条件判断示例
  hosts: webservers
  become: yes
  tasks:
    - name: 检查Nginx配置文件合法性
      command: nginx -t
      register: nginx_config_check
      ignore_errors: yes  # 配置校验失败时不中断执行

    - name: 仅当配置校验通过时，平滑重启Nginx
      service:
        name: nginx
        state: reloaded
      when: nginx_config_check.rc == 0  # 仅当返回码为0时执行

    - name: 配置校验失败时，输出错误信息
      debug:
        msg: "Nginx配置校验失败：{{ nginx_config_check.stderr }}"
      when: nginx_config_check.rc != 0
```

#### 示例 3：捕获结果作为后续任务的参数

将 Register 变量的内容作为后续任务的输入参数，实现动态数据传递：

```yaml
---
- name: Register结果传递示例
  hosts: all
  tasks:
    - name: 获取当前系统内核版本
      command: uname -r
      register: kernel_version
      changed_when: no

    - name: 写入内核版本信息到文件
      copy:
        content: "系统内核版本：{{ kernel_version.stdout }}"
        dest: /tmp/kernel_version.txt
```

### 5.3.4 核心注意事项

1. **作用域限制**：Register 变量仅在当前 Play 内生效，跨 Play 无法直接引用，需通过`hostvars`魔法变量实现跨 Play 传递；
2. **幂等性控制**：用于信息查询的命令 / 模块，需配合`changed_when: no`，避免无意义的 changed 状态标记，符合幂等性要求；
3. **错误处理**：若任务执行失败，默认会中断剧本执行，需配合`ignore_errors: yes`才能捕获失败任务的 Register 变量；
4. **数据类型处理**：`stdout`为字符串类型，若需处理结构化数据（如 JSON），需配合`from_json`过滤器转换为字典 / 列表类型。

---
