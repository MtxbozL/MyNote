
>幂等性是 Ansible Playbook 的核心生命线，是自动化剧本生产环境可用的核心前提。本节基于第一章的幂等性理论，完成 Playbook 场景下的实战落地，覆盖幂等性保障机制、非幂等模块的改造方案、生产环境最佳实践。

### 4.6.1 核心定义回顾

幂等性：一个操作被多次重复执行，其最终系统状态与单次执行的结果完全一致，不会产生任何额外副作用。

- 符合幂等性的 Playbook：首次执行会完成系统状态的收敛，第二次及以后重复执行，不会产生任何系统变更，所有 Task 均为`ok`或`skipped`状态，无`changed`状态；
- 非幂等性的 Playbook：多次执行会重复修改系统状态，产生副作用，如重复写入配置、重复重启服务、重复安装软件，甚至导致系统异常。

### 4.6.2 Playbook 幂等性的核心保障机制

1. **官方模块内置幂等性**：第三章覆盖的所有官方核心模块（yum、copy、file、service、user、firewalld 等）均内置**前置状态校验逻辑**，执行前先对比被控节点当前状态与目标状态，仅当不一致时才执行变更，多次执行无副作用，这是 Playbook 幂等性的核心基础；
2. **自定义幂等性控制**：对于无内置幂等性的模块（command、shell、script），通过 Ansible 提供的标准化机制，手动实现幂等性控制。

### 4.6.3 非幂等模块的幂等性实战方案

command、shell、script 模块无内置幂等性，是 Playbook 幂等性失控的核心风险点，本节提供 4 种标准化的幂等性改造方案，覆盖全场景需求。

#### 方案 1：使用 creates/removes 参数实现基础幂等性

通过`creates`/`removes`参数，定义文件判断条件，仅当条件满足时才执行命令，是最简单的幂等性实现方案，与第三章 Ad-Hoc 用法完全一致。

```yaml
---
- name: creates参数实现幂等性示例
  hosts: all
  become: yes
  tasks:
    # 仅当/data/.initialized标记文件不存在时，才执行初始化脚本
    - name: 执行系统初始化脚本
      shell: /opt/scripts/system_init.sh
      args:
        creates: /data/.initialized

    # 仅当/var/run/nginx.pid文件存在时，才执行平滑重启命令
    - name: 平滑重启Nginx
      shell: nginx -s reload
      args:
        removes: /var/run/nginx.pid
```

#### 方案 2：使用 changed_when 控制变更状态

通过`changed_when`参数，基于命令的返回码、输出内容，自定义 Task 的`changed`状态，避免无意义的变更标记，同时实现幂等性控制。

```yaml
---
- name: changed_when实现幂等性示例
  hosts: all
  become: yes
  tasks:
    # 仅当内核参数实际修改时，才标记为changed
    - name: 开启IPv4转发
      shell: sysctl -w net.ipv4.ip_forward=1
      register: sysctl_result
      changed_when: "'net.ipv4.ip_forward = 1' in sysctl_result.stdout"

    # 命令执行永远不标记为changed，仅用于信息查询
    - name: 查看系统负载
      shell: uptime
      register: uptime_result
      changed_when: no
```

#### 方案 3：使用 when 条件判断实现精准控制

通过`when`条件语句，结合 Facts、注册变量，判断任务是否需要执行，仅当条件满足时才运行命令，是最灵活的幂等性实现方案，第六章会详细讲解条件判断语法。

```yaml
---
- name: when条件实现幂等性示例
  hosts: all
  become: yes
  tasks:
    # 先检查Nginx是否已安装
    - name: 检查Nginx安装状态
      command: which nginx
      register: nginx_check
      ignore_errors: yes
      changed_when: no

    # 仅当Nginx未安装时，才执行编译安装
    - name: 编译安装Nginx
      shell: |
        wget https://nginx.org/download/nginx-1.24.0.tar.gz
        tar -zxf nginx-1.24.0.tar.gz
        cd nginx-1.24.0
        ./configure --prefix=/usr/local/nginx
        make && make install
      when: nginx_check.rc != 0
```

#### 方案 4：标记文件实现一次性任务幂等性

对于初始化、编译安装等一次性执行的任务，执行完成后创建标记文件，后续执行时判断标记文件存在则跳过任务，是生产环境最常用的一次性任务幂等性方案。

```yaml
---
- name: 标记文件实现一次性任务幂等性
  hosts: all
  become: yes
  tasks:
    - name: 检查初始化标记文件
      stat:
        path: /data/.mysql_init_done
      register: init_mark

    - name: 执行MySQL初始化脚本
      shell: /opt/scripts/mysql_init.sh
      when: not init_mark.stat.exists

    - name: 创建初始化完成标记文件
      file:
        path: /data/.mysql_init_done
        state: touch
        mode: 0644
      when: not init_mark.stat.exists
```

### 4.6.4 幂等性生产环境最佳实践

1. **优先使用官方内置幂等模块**：能使用 yum 模块就不用 shell 执行 yum install，能使用 file 模块就不用 shell 执行 mkdir/chmod，能使用 lineinfile 模块就不用 echo >> 追加内容，从根源保障幂等性；
2. **零容忍非受控非幂等模块**：生产环境禁止使用无任何幂等性控制的 command/shell/script 模块，每个非幂等模块必须添加对应的幂等性控制逻辑；
3. **强制重复执行验证**：每个 Playbook 编写完成后，必须至少连续执行 2 次，验证幂等性：首次执行的 changed 任务，第二次执行必须全部为 ok/skipped，无任何 changed 状态；
4. **配合干跑模式验证**：使用`--check`参数模拟执行，验证变更预判的准确性，确保只有需要变更的 Task 才会标记为 changed；
5. **避免增量写入与随机操作**：禁止使用`echo "内容" >> 文件`这类增量写入命令，多次执行会重复写入内容，必须使用 copy、template、lineinfile 等幂等模块替代；禁止在 Playbook 中使用生成随机结果、时间戳的操作，避免每次执行都产生变更。

---

## 本章小结

本章完整覆盖了 Ansible Playbook 的核心语法体系，从 YAML 语法的 Ansible 适配规范、Playbook 四层核心层级结构，到首个标准化剧本的编写与解析、全场景提权机制的层级配置、全链路执行调试工具的使用，最终完成了幂等性的实战落地与生产环境最佳实践。

本章内容是 Ansible 从单任务 Ad-Hoc 操作，进阶到复杂自动化流程编排的核心转折点，是后续变量、流程控制、角色化等高级能力的前置基础，掌握本章内容，即可完成绝大多数标准化业务场景的自动化剧本编写。