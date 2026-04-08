### 9.3.1 场景核心目标

本场景针对企业安全运维的两大核心需求：**常规系统安全合规加固**与**紧急漏洞补丁批量下发**，通过 Ansible 实现批量、可控、可审计的安全运维操作，解决漏洞爆发时的批量修复效率低、合规加固不统一、操作不可追溯的问题。场景覆盖大纲指定的「针对特定漏洞批量升级 OpenSSH 版本」核心需求，同时适配等保合规的系统安全加固。

### 9.3.2 场景 1：特定漏洞批量补丁下发（以 OpenSSH 漏洞修复为例）

针对 OpenSSH 高危漏洞（如 CVE-2024-6387），实现批量版本升级、漏洞修复，同时保障操作的可控性、可回滚、可审计。

#### 核心实现 Playbook：openssh_patch_update.yml

```yaml
---
- name: OpenSSH高危漏洞批量修复 - 版本升级
  hosts: all
  become: yes
  gather_facts: yes
  serial: 20%  # 分批执行，每批20%节点，避免全量业务中断
  vars:
    # 漏洞修复的目标最低版本
    target_openssh_version: "8.7p1"
    # 升级前是否备份配置
    backup_config: yes
    # 升级后是否重启sshd服务
    restart_sshd: yes

  pre_tasks:
    - name: 获取当前OpenSSH版本
      command: ssh -V
      register: openssh_version_output
      changed_when: no
      failed_when: openssh_version_output.rc != 0

    - name: 解析当前OpenSSH版本号
      set_fact:
        current_openssh_version: "{{ openssh_version_output.stderr.split(',')[0].split('_')[1].split('p')[0] }}.{{ openssh_version_output.stderr.split(',')[0].split('p')[1] }}"
      when: openssh_version_output is succeeded

    - name: 输出当前版本信息
      debug:
        msg: "当前节点 {{ inventory_hostname }} OpenSSH版本：{{ current_openssh_version }}，目标版本：{{ target_openssh_version }}"

    - name: 备份OpenSSH配置文件
      archive:
        path: /etc/ssh
        dest: /opt/backup/ssh_config_backup_{{ ansible_date_time.date }}.tar.gz
        mode: 0600
      when: backup_config | bool and current_openssh_version is version(target_openssh_version, '<')

  tasks:
    - name: 升级OpenSSH至目标版本
      yum:
        name: openssh
        state: latest
        enablerepo: "{{ security_repo | default('updates') }}"
      when: current_openssh_version is version(target_openssh_version, '<')
      notify: 重启sshd服务

  post_tasks:
    - name: 验证升级后的OpenSSH版本
      command: ssh -V
      register: final_openssh_version
      changed_when: no

    - name: 输出升级结果
      debug:
        msg: "节点 {{ inventory_hostname }} 升级完成，最终版本：{{ final_openssh_version.stderr }}"

    - name: 升级失败告警
      fail:
        msg: "节点 {{ inventory_hostname }} 升级失败，版本未达到目标要求"
      when: final_openssh_version is succeeded and current_openssh_version is version(target_openssh_version, '<') and final_openssh_version.stderr is not search(target_openssh_version)

  handlers:
    - name: 重启sshd服务
      service:
        name: sshd
        state: restarted
      when: restart_sshd | bool
```

### 9.3.3 场景 2：系统安全合规加固

针对等保 2.0 三级合规要求，实现系统全维度安全加固，覆盖账号安全、权限控制、密码策略、日志审计、服务禁用、内核安全参数等核心合规项。

#### 核心加固任务示例：security_hardening.yml

```yaml
---
- name: 企业级系统安全合规加固
  hosts: all
  become: yes
  gather_facts: yes
  vars:
    max_password_age: 90
    min_password_length: 12
    lockout_attempts: 5
    lockout_time: 900

  tasks:
    - name: 配置系统密码策略 - 密码最长有效期
      lineinfile:
        path: /etc/login.defs
        regexp: "^PASS_MAX_DAYS"
        line: "PASS_MAX_DAYS {{ max_password_age }}"
        state: present

    - name: 配置系统密码策略 - 最小密码长度
      lineinfile:
        path: /etc/security/pwquality.conf
        regexp: "^minlen"
        line: "minlen = {{ min_password_length }}"
        state: present

    - name: 配置登录失败锁定策略
      blockinfile:
        path: /etc/pam.d/system-auth
        block: |
          auth        required      pam_faillock.so preauth silent audit deny={{ lockout_attempts }} unlock_time={{ lockout_time }}
          auth        [default=die] pam_faillock.so authfail audit deny={{ lockout_attempts }} unlock_time={{ lockout_time }}
          account     required      pam_faillock.so
        state: present

    - name: 禁用SSH密码认证（仅密钥登录）
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "^PasswordAuthentication"
        line: "PasswordAuthentication no"
        state: present
      notify: 重启sshd服务
      when: disable_ssh_password_auth | default(true) | bool

    - name: 禁用SSH root用户远程登录
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "^PermitRootLogin"
        line: "PermitRootLogin no"
        state: present
      notify: 重启sshd服务

    - name: 禁用所有不必要的SUID/SGID权限文件
      file:
        path: "{{ item }}"
        mode: 'u-s,g-s'
      loop: "{{ dangerous_suid_files }}"
      ignore_errors: yes

    - name: 开启系统审计服务
      service:
        name: auditd
        state: started
        enabled: yes

    - name: 配置审计规则
      copy:
        src: files/audit.rules
        dest: /etc/audit/rules.d/hardening.rules
        mode: 0600
      notify: 重载审计规则

  handlers:
    - name: 重启sshd服务
      service:
        name: sshd
        state: restarted

    - name: 重载审计规则
      command: augenrules --load
      changed_when: yes
```

### 9.3.4 最佳实践与避坑指南

1. **分批执行控制**：高危补丁下发、SSH 配置修改等操作，必须通过`serial`参数分批执行，先小范围灰度验证，再全量推广，避免全量业务中断；
2. **前置备份机制**：所有修改系统配置、升级核心组件的操作，必须先备份原配置 / 文件，预留回滚方案；
3. **版本校验闭环**：补丁升级后必须添加版本校验步骤，确认漏洞修复完成，避免升级成功但版本未更新的假成功问题；
4. **干跑预验证**：所有加固操作执行前，必须通过`--check`参数干跑，确认变更范围，避免非预期的系统修改；
5. **防锁定保障**：SSH 加固、防火墙配置等操作，必须先验证备用登录通道正常，避免批量锁定服务器；
6. **合规可审计**：所有操作必须保留完整执行日志，通过 Callback 插件上报审计平台，满足等保合规的审计要求；
7. **回滚预案**：针对每个补丁下发、加固操作，必须配套对应的回滚 Playbook，出现异常时可快速恢复。

---
