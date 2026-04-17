
>动态 Inventory 是 Ansible 适配动态变化基础设施的核心能力，为可执行的脚本 / 插件，Ansible 执行时自动调用并返回符合规范的 JSON 格式主机清单，无需手动维护静态文件。

### 2.6.1 核心定义与适用场景

**核心定义**：动态 Inventory 是符合 Ansible 接口规范的可执行程序，通过 API 对接外部系统，实时拉取主机资产数据，动态生成标准化的主机清单、分组与变量，实现基础设施资产的自动化同步。

**核心适用场景**：

1. 公有云 / 私有云环境：AWS、阿里云、腾讯云等，云主机动态创建 / 销毁，静态清单无法实时同步；
2. 企业 CMDB 系统：资产元数据统一存储在 CMDB，动态 Inventory 实现资产数据的统一管理与消费；
3. 容器 / 虚拟化平台：Kubernetes、VMware vSphere、OpenStack 等，资源动态调度，地址与拓扑频繁变化；
4. 超大规模集群：数千台以上节点，静态清单维护成本极高，动态 Inventory 实现自动化纳管。


```json
{
  "ansible_host": "192.168.1.101",
  "ansible_port": 2222,
  "env": "production",
  "business": "web",
  "region": "china"
}
```
    

### 2.6.2 标准化接口规范

Ansible 对动态 Inventory 程序有严格的接口要求，必须支持两个核心参数，返回符合 JSON Schema 规范的结果：

1. **--list 参数**：必须返回完整的主机分组、组变量、主机变量、嵌套关系的 JSON 结构，必须包含`all`与`ungrouped`内置组，是核心调用参数。
    
    最小化合法返回示例：


    ```json
    {
      "all": {
        "hosts": ["192.168.1.100", "192.168.1.101", "192.168.1.201"]
      },
      "ungrouped": {
        "hosts": ["192.168.1.100"]
      },
      "webservers": {
        "hosts": ["192.168.1.101"],
        "vars": {
          "ansible_user": "ops",
          "ansible_become": "yes"
        }
      },
      "dbservers": {
        "hosts": ["192.168.1.201"]
      },
      "production": {
        "children": ["webservers", "dbservers"]
      }
    }
    ```

2. **--host <"hostname"> 参数**：必须返回指定主机的专属变量 JSON 对象，无变量则返回空对象`{}`，用于主机级变量的精准查询。

```json
{
  "ansible_host": "192.168.1.101",
  "ansible_port": 2222,
  "env": "production",
  "business": "web",
  "region": "china"
}
```

### 2.6.3 动态 Inventory 使用方式

1. **权限要求**：动态 Inventory 程序必须具备可执行权限（`chmod +x inventory_script.py`），Ansible 方可调用执行；
2. **调用方式**：与静态 Inventory 完全一致，通过`-i`参数指定动态 Inventory 程序路径：
    
    ```bash
    # 使用动态Inventory执行连通性测试
    ansible all -i inventory_script.py -m ping
    
    # 使用动态Inventory执行Playbook
    ansible-playbook -i inventory_script.py playbook.yml
    ```
    
3. **静态与动态混合使用**：Ansible 支持将多个静态文件、动态程序放在同一目录下，执行时指定该目录，Ansible 会自动合并所有 Inventory 内容，实现固定资产与动态资产的统一管理：
    
    ```bash
    # 目录结构
    inventory/
    ├── static_hosts    # 静态Inventory文件
    └── aliyun_dynamic.py  # 阿里云动态Inventory脚本
    
    # 执行时指定目录，自动合并
    ansible all -i inventory/ -m ping
    ```
    

### 2.6.4 主流官方动态 Inventory 插件

Ansible 官方与云厂商提供了成熟的动态 Inventory 插件，无需自行开发，基于 YAML 配置文件即可快速对接，主流插件包括：

- AWS EC2：`amazon.aws.aws_ec2` 插件
- 阿里云：`alibaba.cloud.alicloud` 插件
- 腾讯云：`tencentcloud.tencentcloud.tencentcloud` 插件
- 华为云：`huaweicloud.huaweicloud.huaweicloud` 插件
- VMware vSphere：`community.vmware.vmware_vm_inventory` 插件

### 2.6.5 自定义动态 Inventory 开发入门

基于 Python 实现最小化动态 Inventory 脚本，可快速对接企业内部 CMDB、资产管理系统，核心示例如下：

```python
#!/usr/bin/env python3
import json
import sys

# 生产环境替换为从CMDB/云API拉取的实时资产数据
INVENTORY_DATA = {
    "all": {
        "hosts": ["192.168.1.101", "192.168.1.102", "192.168.1.201"],
        "vars": {
            "ansible_user": "ops",
            "ansible_become": "yes"
        }
    },
    "webservers": {
        "hosts": ["192.168.1.101", "192.168.1.102"],
        "vars": {"nginx_version": "1.24.0"}
    },
    "dbservers": {
        "hosts": ["192.168.1.201"],
        "vars": {"mysql_version": "8.0"}
    }
}

# 接口参数处理
if __name__ == "__main__":
    if len(sys.argv) == 2 and sys.argv[1] == "--list":
        print(json.dumps(INVENTORY_DATA, indent=2))
    elif len(sys.argv) == 3 and sys.argv[1] == "--host":
        # 生产环境替换为从CMDB拉取指定主机的变量
        print(json.dumps({}, indent=2))
    else:
        print("Usage: %s --list | --host <hostname>" % sys.argv[0])
        sys.exit(1)
```

---
