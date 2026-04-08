### 7.5.1 核心定义

Ansible Galaxy 是 Ansible 官方维护的全球最大的 Roles 角色开源社区，是 Ansible 生态的核心组成部分，地址为：[https://galaxy.ansible.com](https://galaxy.ansible.com)。社区汇聚了全球开发者贡献的海量成熟、标准化的 Roles 角色，覆盖系统初始化、中间件部署、数据库安装、监控配置、云资源管理等全场景，可直接下载使用，无需从零开发自动化逻辑，大幅提升开发效率。

### 7.5.2 ansible-galaxy 命令行工具

Ansible 内置`ansible-galaxy`命令行工具，是与 Ansible Galaxy 社区交互的核心工具，覆盖角色的搜索、下载、安装、管理全流程。

#### 1. 角色搜索：ansible-galaxy search

用于在 Galaxy 社区中搜索符合关键词的角色，提前筛选合适的开源角色。

```bash
# 基础搜索：搜索nginx相关角色
ansible-galaxy search nginx

# 按作者筛选：搜索Geerlingguy贡献的角色（大纲指定的核心优质作者）
ansible-galaxy search --author geerlingguy

# 按平台筛选：搜索兼容EL8的mysql角色
ansible-galaxy search mysql --platform EL --platform-version 8

# 完整参数帮助
ansible-galaxy search --help
```

> 核心说明：Jeff Geerling 是 Ansible 官方认证的顶级贡献者，其贡献的角色（nginx、mysql、redis、php 等）质量极高、更新及时、兼容性强，是生产环境的首选开源角色。

#### 2. 角色安装：ansible-galaxy install

用于从 Galaxy 社区下载并安装角色到本地，是最核心的常用命令。

```bash
# 1. 基础安装：安装最新版角色
ansible-galaxy install geerlingguy.nginx

# 2. 安装指定版本的角色
ansible-galaxy install geerlingguy.nginx,3.2.0

# 3. 指定安装路径
ansible-galaxy install geerlingguy.nginx -p ./roles/

# 4. 批量安装：通过requirements.yml文件批量安装角色
ansible-galaxy install -r requirements.yml

# 5. 强制覆盖已安装的同名角色
ansible-galaxy install geerlingguy.nginx --force
```

#### 3. 批量安装 requirements.yml 规范

生产环境推荐使用`requirements.yml`文件管理所有依赖的角色，实现版本锁定、批量安装，便于版本控制与团队协作。

**标准示例**：`requirements.yml`

```yaml
---
roles:
  # 从Galaxy社区安装指定版本的nginx角色
  - name: geerlingguy.nginx
    version: 3.2.0
  # 安装mysql角色
  - name: geerlingguy.mysql
    version: 4.4.0
  # 从Git仓库安装自定义角色
  - name: custom-nginx
    src: https://github.com/example/custom-nginx.git
    version: main
    scm: git
  # 从本地压缩包安装
  - name: local-role
    src: file:///opt/roles/local-role.tar.gz
```

**执行批量安装**：

```bash
ansible-galaxy install -r requirements.yml -p ./roles/
```

#### 4. 其他核心命令

```bash
# 列出本地已安装的所有角色
ansible-galaxy list

# 查看已安装角色的详细信息
ansible-galaxy info geerlingguy.nginx

# 卸载已安装的角色
ansible-galaxy remove geerlingguy.nginx

# 登录Galaxy社区，用于上传自己的角色
ansible-galaxy login
```

### 7.5.3 开源角色使用最佳实践与安全规范

1. **版本锁定**：生产环境必须锁定角色的版本号，禁止安装 latest 版本，避免上游更新导致的兼容性问题；
2. **优先官方认证角色**：优先选择 Ansible 官方认证、高下载量、高 Star、活跃更新的角色，优先使用 Geerlingguy 等顶级贡献者的角色；
3. **代码审计**：生产环境使用前，必须审计角色的代码逻辑，避免恶意代码、不安全配置、不符合企业规范的操作；
4. **本地缓存**：企业级场景推荐将角色同步到内部 Git 仓库，避免依赖外部网络，同时保障代码可控；
5. **变量覆盖**：通过 defaults 变量的优先级特性，使用 Inventory 变量、Play 变量覆盖角色的默认配置，禁止直接修改角色源码，便于后续版本升级；
6. **测试验证**：使用前必须在测试环境完整验证角色的功能、兼容性、幂等性，确认符合生产环境要求。

---
