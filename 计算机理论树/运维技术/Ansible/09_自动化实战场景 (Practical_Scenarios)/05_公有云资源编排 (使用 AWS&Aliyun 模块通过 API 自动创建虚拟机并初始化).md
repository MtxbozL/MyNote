### 9.5.1 场景核心目标

本场景通过 Ansible 的云厂商模块，实现公有云资源的全生命周期自动化编排，覆盖从云服务器、网络、安全组的创建，到新创建资源的自动初始化全流程，完全覆盖大纲指定的「使用 AWS/Aliyun 模块通过 API 自动创建虚拟机并初始化」核心需求。本场景以国内主流的阿里云为例，同时兼容 AWS，实现「基础设施即代码」的云资源自动化管理。

### 9.5.2 前置依赖与环境准备

1. **安装阿里云 Ansible 集合**：

    ```bash
    ansible-galaxy collection install alibaba.cloud
    ```
    
2. **阿里云 AK/SK 配置**：
    
    - 方式 1：通过环境变量配置`ALIBABA_CLOUD_ACCESS_KEY_ID`与`ALIBABA_CLOUD_ACCESS_KEY_SECRET`；
    - 方式 2：通过 Ansible Vault 加密存储在变量文件中，禁止明文存储；
    
3. **权限要求**：AK 需具备 ECS、VPC、EIP、安全组的全量管理权限。

### 9.5.3 完整云资源编排 Playbook 实现

#### 1. 主 Playbook：aliyun_ecs_manage.yml

```yaml
---
- name: 阿里云ECS资源自动化编排与初始化
  hosts: localhost
  gather_facts: no
  vars_files:
    - ./vars/aliyun_config.yml
    - ./vars/secret.yml  # 加密存储AK/SK
  tasks:
    - name: 1. 创建专有网络VPC
      alibaba.cloud.alicloud_vpc:
        alicloud_access_key: "{{ alicloud_access_key }}"
        alicloud_secret_key: "{{ alicloud_secret_key }}"
        region_id: "{{ region_id }}"
        vpc_name: "{{ vpc_name }}"
        cidr_block: "{{ vpc_cidr }}"
        state: present
      register: vpc_result

    - name: 2. 创建交换机
      alibaba.cloud.alicloud_vswitch:
        alicloud_access_key: "{{ alicloud_access_key }}"
        alicloud_secret_key: "{{ alicloud_secret_key }}"
        region_id: "{{ region_id }}"
        vpc_id: "{{ vpc_result.vpc.id }}"
        zone_id: "{{ zone_id }}"
        vswitch_name: "{{ vswitch_name }}"
        cidr_block: "{{ vswitch_cidr }}"
        state: present
      register: vswitch_result

    - name: 3. 创建安全组
      alibaba.cloud.alicloud_security_group:
        alicloud_access_key: "{{ alicloud_access_key }}"
        alicloud_secret_key: "{{ alicloud_secret_key }}"
        region_id: "{{ region_id }}"
        vpc_id: "{{ vpc_result.vpc.id }}"
        name: "{{ security_group_name }}"
        description: "Ansible自动化创建的Web服务安全组"
        state: present
      register: sg_result

    - name: 4. 配置安全组规则
      alibaba.cloud.alicloud_security_group_rule:
        alicloud_access_key: "{{ alicloud_access_key }}"
        alicloud_secret_key: "{{ alicloud_secret_key }}"
        region_id: "{{ region_id }}"
        security_group_id: "{{ sg_result.security_group.id }}"
        type: ingress
        ip_protocol: tcp
        port_range: "{{ item.port_range }}"
        cidr_ip: "{{ item.cidr_ip }}"
        policy: accept
        priority: 1
        state: present
      loop: "{{ security_group_rules }}"

    - name: 5. 创建ECS实例
      alibaba.cloud.alicloud_instance:
        alicloud_access_key: "{{ alicloud_access_key }}"
        alicloud_secret_key: "{{ alicloud_secret_key }}"
        region_id: "{{ region_id }}"
        zone_id: "{{ zone_id }}"
        instance_name: "{{ instance_name }}"
        image_id: "{{ image_id }}"
        instance_type: "{{ instance_type }}"
        security_groups: ["{{ sg_result.security_group.id }}"]
        vswitch_id: "{{ vswitch_result.vswitch.id }}"
        system_disk_category: cloud_essd
        system_disk_size: 40
        internet_max_bandwidth_out: 10
        password: "{{ ecs_root_password }}"
        state: running
        count: "{{ instance_count }}"
      register: ecs_result
      no_log: yes

    - name: 6. 分配并绑定弹性公网IP
      alibaba.cloud.alicloud_eip:
        alicloud_access_key: "{{ alicloud_access_key }}"
        alicloud_secret_key: "{{ alicloud_secret_key }}"
        region_id: "{{ region_id }}"
        bandwidth: 10
        internet_charge_type: PayByTraffic
        instance_id: "{{ item.id }}"
        state: present
      loop: "{{ ecs_result.instances }}"
      register: eip_result
      when: assign_eip | bool

    - name: 7. 输出ECS实例信息
      debug:
        msg: "ECS实例创建成功，实例ID：{{ item.id }}，公网IP：{{ item.public_ip_address }}，私网IP：{{ item.private_ip_address }}"
      loop: "{{ ecs_result.instances }}"

    - name: 8. 动态生成Inventory主机清单
      template:
        src: hosts.j2
        dest: ./inventory/cloud_ecs_hosts
        mode: 0644

    - name: 9. 等待ECS实例SSH服务就绪
      wait_for:
        host: "{{ item.public_ip_address }}"
        port: 22
        state: started
        timeout: 300
        delay: 30
      loop: "{{ ecs_result.instances }}"
      when: wait_for_ssh | bool

- name: 10. 新创建ECS实例基线初始化
  hosts: cloud_ecs
  become: yes
  gather_facts: yes
  roles:
    - role: repo_manage
    - role: base_packages
    - role: security_base
    - role: nginx
```

#### 2. 变量配置文件：vars/aliyun_config.yml

```yaml
---
# 地域与可用区
region_id: "cn-shenzhen"
zone_id: "cn-shenzhen-a"

# 网络配置
vpc_name: "ansible-vpc"
vpc_cidr: "192.168.0.0/16"
vswitch_name: "ansible-vswitch"
vswitch_cidr: "192.168.1.0/24"

# 安全组配置
security_group_name: "ansible-web-sg"
security_group_rules:
  - port_range: "22/22"
    cidr_ip: "0.0.0.0/0"
  - port_range: "80/80"
    cidr_ip: "0.0.0.0/0"
  - port_range: "443/443"
    cidr_ip: "0.0.0.0/0"

# ECS实例配置
instance_name: "web-server"
instance_count: 2
instance_type: "ecs.c7.large"
image_id: "aliyun_3_x64_20G_alibase_20240319.vhd"
assign_eip: yes
wait_for_ssh: yes
```

### 9.5.4 最佳实践与避坑指南

1. **敏感信息安全**：云厂商 AK/SK、ECS 密码等敏感信息，必须通过 Ansible Vault 加密，禁止明文提交到 Git 仓库；
2. **幂等性保障**：所有云资源操作均采用声明式模块，重复执行不会重复创建资源，仅当配置变更时才会修改资源；
3. **最小权限原则**：云 AK 需遵循最小权限原则，仅分配所需的资源管理权限，禁止使用管理员全权限 AK；
4. **网络安全配置**：安全组规则需严格限制端口与源 IP，禁止 0.0.0.0/0 全放开高危端口（如 22），生产环境需限制为企业办公网 IP；
5. **资源标签管理**：创建云资源时必须添加业务、环境、负责人等标签，便于成本核算、资源管理与审计；
6. **状态管理**：大规模云资源编排推荐使用 Ansible Tower/AWX 进行状态管理，避免多人员同时操作导致的资源冲突；
7. **销毁机制**：配套资源销毁 Playbook，通过`state: absent`实现资源的一键清理，避免测试资源闲置产生成本；
8. **多环境隔离**：通过变量区分开发、测试、生产环境，不同环境使用独立的 VPC 与资源栈，避免环境混部。

---

## 本章小结

本章完整覆盖了 Ansible 企业级落地的五大核心实战场景，从服务器上线基线初始化、经典 Web 架构一键部署、安全漏洞批量修复、数据灾备全流程自动化，到公有云资源编排，将前序八章的所有理论知识完整落地到生产环境。每个场景均遵循 Ansible 工程化最佳实践，严格保障幂等性、安全性、可复用性，所有代码均可直接适配企业生产环境，实现了从理论知识到自动化运维能力的完整闭环。