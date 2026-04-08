## 2.3 主机分组与嵌套

主机分组是 Ansible 实现批量自动化管理的核心能力，通过对被控节点按业务线、环境、地域、功能等维度进行逻辑分组，可实现精准的批量操作，避免单节点重复配置。

### 2.3.1 分组核心规则

1. **多归属规则**：一台主机可同时归属多个自定义分组，无数量限制，支持多维度标签化管理；
2. **命名规范**：组名仅可包含字母、数字、下划线，不可以数字开头，不可包含空格、特殊字符（短横线`-`兼容但不推荐，避免与变量名冲突）；
3. **继承规则**：子组的所有主机自动归属父组，同时继承父组的所有变量，主机级变量优先级高于组级，子组变量优先级高于父组；
4. **嵌套规则**：分组嵌套无深度限制，生产环境建议不超过 3 层，避免管理复杂度失控。

### 2.3.2 分组嵌套（Children 组）

分组嵌套即定义「组的组」，通过父组包含子组的方式，实现多层级的基础设施拓扑管理，适配企业级多环境、多地域的集群架构。

#### INI 格式嵌套语法

通过 `[父组名:children]` 定义嵌套关系，父组包含指定的子组：

```ini
# 按功能维度定义基础分组
[webservers]
192.168.1.[101:102]
[dbservers]
192.168.1.[201:202]
[cacheservers]
192.168.1.[301:302]

# 按环境维度定义父组，嵌套功能子组
[production:children]
webservers
dbservers
cacheservers

[test:children]
webservers
dbservers

# 按地域维度定义顶级父组，嵌套环境子组
[china_region:children]
production
test

# 父组同时支持直接定义主机与嵌套子组
[monitoring:children]
production
[monitoring:hosts]
192.168.1.250
```

上述示例中，`china_region` 顶级组自动包含 `production` 与 `test` 子组的所有主机，可直接通过 `ansible china_region -m ping` 对全地域节点执行操作。

#### YAML 格式嵌套语法

通过 `children` 字段定义嵌套关系，层级结构更直观，与上述 INI 格式完全对等的 YAML 结构如下：

```yaml
---
all:
  children:
    china_region:
      children:
        production:
          children:
            webservers:
              hosts:
                192.168.1.[101:102]:
            dbservers:
              hosts:
                192.168.1.[201:202]:
            cacheservers:
              hosts:
                192.168.1.[301:302]:
        test:
          children:
            webservers:
              hosts:
                192.168.1.[101:102]:
            dbservers:
              hosts:
                192.168.1.[201:202]:
    monitoring:
      children:
        production:
      hosts:
        192.168.1.250:
```

---
