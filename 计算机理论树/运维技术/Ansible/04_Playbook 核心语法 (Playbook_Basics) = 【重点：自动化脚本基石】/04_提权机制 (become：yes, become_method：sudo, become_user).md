Ansible 的 Become 提权机制，是通过系统特权升级工具，将远程普通用户的权限提升至目标用户（通常为 root），执行高权限操作的核心能力。是 Linux 系统批量自动化的基础，避免直接使用 root 用户远程登录，符合安全合规要求，与第一章、第二章的提权配置形成完整体系。

### 4.4.1 提权配置的层级与优先级

Ansible 提权配置遵循**就近原则**，层级越具体，优先级越高，从高到低的优先级顺序为：

1. **Task 级提权配置**：仅对当前 Task 生效，优先级最高，可覆盖 Play 级、全局级配置；
2. **Play 级提权配置**：对当前 Play 内的所有 Task 生效，可覆盖 Inventory 级、全局级配置；
3. **Inventory 主机 / 组级提权配置**：对指定主机 / 组生效，对应第二章的`ansible_become`系列参数；
4. **ansible.cfg 全局配置**：对所有 Playbook、Ad-Hoc 命令生效，优先级最低。

### 4.4.2 全场景配置示例

#### 1. Play 级全局提权配置

对当前 Play 内的所有 Task 生效，是最常用的配置方式：

```yaml
---
- name: Play级全局提权示例
  hosts: all
  remote_user: ops
  # 全局开启提权
  become: yes
  # 提权方式为sudo
  become_method: sudo
  # 提权至root用户
  become_user: root
  tasks:
    # 所有Task默认继承全局提权配置
    - name: 安装系统依赖
      yum:
        name: ['git', 'wget', 'htop']
        state: present
```

#### 2. Task 级精细化提权配置

仅对当前 Task 生效，遵循最小权限原则，仅在需要高权限的 Task 开启提权：

```yaml
---
- name: Task级精细化提权示例
  hosts: all
  remote_user: ops
  # 全局关闭提权
  become: no
  tasks:
    # 无需提权的任务，以普通用户执行
    - name: 拉取应用代码
      git:
        repo: https://github.com/example/app.git
        dest: /home/ops/app/
        version: main

    # 需要高权限的任务，单独开启提权
    - name: 安装应用依赖
      yum:
        name: ['python3', 'python3-pip']
        state: present
      become: yes
```

#### 3. 提权密码配置

生产环境**严禁在 Playbook/Inventory 中明文写入提权密码**，仅可通过以下两种安全方式传递：

1. 执行时交互式输入密码：通过`-K/--ask-become-pass`参数，执行时提示输入提权密码：

    ```bash
    ansible-playbook nginx_deploy.yml -K
    ```
    
2. 加密存储：通过 Ansible Vault 加密提权密码，第八章会详细讲解。

### 4.4.3 提权最佳实践

1. **最小权限原则**：优先使用 Task 级提权，仅在需要高权限的任务开启提权，而非全局开启；
2. **免密 sudo 配置**：生产环境推荐配置被控节点普通用户的 sudo 免密权限，避免执行时手动输入密码，适配批量自动化场景，配置方式见第二章；
3. **禁止明文密码**：绝对禁止在 Playbook、Inventory 文件中明文写入连接密码或提权密码，必须通过 Ansible Vault 加密；
4. **禁止 root 远程登录**：遵循安全合规要求，禁止直接使用 root 用户远程连接，使用普通用户 + sudo 提权的方式执行高权限操作；
5. **提权范围控制**：通过 sudoers 配置，限制普通用户仅能执行特定命令的提权，而非全量 root 权限，进一步缩小攻击面。

---
