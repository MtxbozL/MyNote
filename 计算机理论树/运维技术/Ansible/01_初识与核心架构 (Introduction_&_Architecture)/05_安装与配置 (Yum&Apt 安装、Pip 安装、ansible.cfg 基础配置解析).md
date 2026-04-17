### 1.5.1 安装前置要求

#### 控制节点前置要求

- 操作系统：Linux（RHEL/CentOS/Rocky Linux/Debian/Ubuntu/Fedora）、macOS、Windows WSL2；
- Python 环境：Python 3.8 及以上版本；
- 网络要求：可访问官方软件源 / PyPI 源，与被控节点的 SSH/WinRM 端口网络可达。

#### 被控节点前置要求

- Linux/Unix 系统：开启 SSH 服务，预装 Python 2.7/3.5 及以上版本；
- Windows 系统：开启 WinRM 服务，预装 PowerShell 3.0 及以上版本。

### 1.5.2 主流安装方式

#### 1.5.2.1 系统包管理器安装（RPM 系 - Yum/Dnf）

适用于 RHEL、CentOS、Rocky Linux、AlmaLinux、Fedora 等 RPM 系发行版，是生产环境的首选稳定安装方式。

```bash
# 1. 安装EPEL软件源（Ansible官方包位于EPEL源）
sudo yum install -y epel-release
sudo yum makecache

# 2. 安装Ansible核心程序
sudo yum install -y ansible

# 3. 验证安装
ansible --version
```

- 优势：安装简单，依托系统包管理器完成依赖与版本管理，稳定性强；
- 局限：软件源中的版本通常滞后于官方最新稳定版。

#### 1.5.2.2 系统包管理器安装（Debian 系 - Apt）

适用于 Debian、Ubuntu 等 Debian 系发行版。

```bash
# 1. 更新软件源缓存，安装依赖
sudo apt update
sudo apt install -y software-properties-common

# 2. 添加Ansible官方PPA源（获取最新稳定版，可选）
sudo add-apt-repository --yes --update ppa:ansible/ansible

# 3. 安装Ansible
sudo apt install -y ansible

# 4. 验证安装
ansible --version
```

#### 1.5.2.3 Pip 跨平台安装方式

官方推荐的跨平台安装方式，适用于所有支持 Python 的系统，可灵活选择安装版本，获取官方最新稳定版。

```bash
# 1. 安装Python3与pip包管理器
# RPM系
sudo yum install -y python3 python3-pip
# Debian系
sudo apt install -y python3 python3-pip

# 2. 升级pip至最新版
python3 -m pip install --upgrade pip

# 3. 安装Ansible
# ubuntu 版本需要创建虚拟环境
cd /opt
sudo mkdir /ansible
sudo python3 -m venv ansible_venv
source /opt/ansible/ansible_venv/bin/activate
# 安装最新稳定版
python3 -m pip install ansible
# 安装指定版本
python3 -m pip install ansible==9.2.0

# 4. 验证安装
ansible --version
```

- 优势：跨平台兼容性强，版本选择灵活，可获取最新版；
- 最佳实践：推荐使用 Python venv 虚拟环境安装，避免污染系统 Python 环境。

### 1.5.3 ansible.cfg 核心配置文件解析

`ansible.cfg`是 Ansible 的全局配置文件，采用 INI 格式，控制 Ansible 的所有默认行为，本节明确其加载优先级与核心配置段。

#### 1.5.3.1 配置文件加载优先级

Ansible 按照 **从高到低** 的顺序加载配置文件，**仅加载第一个匹配到的配置文件，后续低优先级文件将被完全忽略**，该规则是避免配置冲突的核心：

1. **环境变量指定的配置文件**：通过`ANSIBLE_CONFIG`环境变量指定的文件路径，优先级最高；
2. **当前工作目录配置文件**：执行 Ansible 命令的当前目录下的`ansible.cfg`文件，适配单项目专属配置；
3. **当前用户家目录配置文件**：`~/.ansible.cfg`，适配当前用户的全局配置；
4. **系统全局配置文件**：`/etc/ansible/ansible.cfg`，Ansible 安装后默认生成的全局配置，优先级最低。

需要注意的是，使用 pip 在虚拟环境中安装的 ansible 默认是不会生成配置文件的：

![[Pasted image 20260415171946.png]]

#### 🔍 为什么会这样？（两种安装方式的核心区别）

| 安装方式                          | 是否自动生成全局配置 | 说明                                                               |
| ----------------------------- | ---------- | ---------------------------------------------------------------- |
| `apt install ansible`（系统包安装）  | ✅ 自动生成     | 会自动创建 `/etc/ansible/` 目录、全局 `ansible.cfg` 配置文件、`hosts` 主机清单，开箱即用 |
| `pip install ansible`（虚拟环境安装） | ❌ 不会生成     | 仅安装纯 Python 代码，不修改系统目录，不会创建任何配置文件，完全依赖手动配置                       |

当前的状态就是后者：Ansible 本身安装成功（`ansible [core 2.20.4]` 正常显示），但因为没有找到任何配置文件，所以用的是 **硬编码的默认参数**，不影响核心功能，只是每次执行命令需要手动指定参数（比如 `-i` 清单、`-u` 远程用户）。

因此需要自行创建配置文件：

#### 🛠️ 两种配置方案（按需选择）

##### 方案 1：创建系统全局配置（推荐，服务器 / 运维场景）

适合需要统一配置、所有用户都能使用的场景，创建后 `ansible --version` 会显示配置文件路径：

```bash
# 1. 创建全局配置目录
sudo mkdir -p /etc/ansible

# 2. 生成基础全局配置文件 ansible.cfg
sudo tee /etc/ansible/ansible.cfg << 'EOF'
[defaults]
# 主机清单文件路径
inventory = /etc/ansible/hosts
# 关闭SSH首次连接的主机密钥确认
host_key_checking = False
# 远程连接默认用户（根据你的系统修改，这里用ubuntu）
remote_user = ubuntu
# 日志文件路径
log_path = /var/log/ansible.log
# SSH连接超时时间
timeout = 10

[privilege_escalation]
# 默认提权为root
become = True
become_method = sudo
become_user = root
EOF

# 3. 创建主机清单文件
sudo touch /etc/ansible/hosts

# 4. 给日志文件授权（避免写入报错）
sudo touch /var/log/ansible.log && sudo chmod 666 /var/log/ansible.log
```

执行完成后，再次运行 `ansible --version`，就会显示：

```shell
config file = /etc/ansible/ansible.cfg
```

✅ 全局配置生效。

---

##### 方案 2：创建用户级 / 项目级配置（个人开发场景）

如果不想修改系统目录，仅为当前用户 / 项目配置：

- **用户级（对当前 ubuntu 用户所有项目生效）**

```bash
# 在家目录创建用户级配置
tee ~/.ansible.cfg << 'EOF'
[defaults]
inventory = ~/.ansible/hosts
host_key_checking = False
remote_user = ubuntu
log_path = ~/.ansible/ansible.log
EOF
# 创建用户级主机清单
mkdir -p ~/.ansible && touch ~/.ansible/hosts
```

- **项目级（仅对当前项目目录生效，优先级最高）**
    
    在你的 Ansible 项目文件夹中，直接创建 `ansible.cfg` 文件，写入对应配置即可，Ansible 会优先使用这个项目配置。

#### 1.5.3.2 核心配置段与关键参数

`ansible.cfg`分为多个配置段，核心基础配置段与关键参数如下，覆盖所有基础运行所需配置：

1. **[defaults] 段：全局默认核心配置**

| 参数                | 作用              | 默认值                | 基础配置建议            |
| :---------------- | :-------------- | :----------------- | :---------------- |
| inventory         | 默认资产清单文件路径      | /etc/ansible/hosts | 可自定义为项目内 hosts 文件 |
| remote_user       | 默认远程连接用户名       | 当前执行命令的系统用户        | 统一指定被控节点管理用户      |
| become            | 是否默认开启权限提升      | no                 | 生产环境建议按需开启        |
| become_method     | 默认提权方式          | sudo               | 适配目标系统的提权机制       |
| become_user       | 默认提权至的目标用户      | root               | 绝大多数场景保持默认        |
| forks             | 并行执行的最大并发数      | 5                  | 大规模集群可提升至 50+     |
| host_key_checking | 是否开启 SSH 主机密钥检查 | yes                | 测试环境可关闭，跳过首次连接确认  |
| timeout           | SSH 连接超时时间（秒）   | 10                 | 复杂网络环境可适当延长       |

2. **[ssh_connection] 段：SSH 连接性能与稳定性配置**

| 参数           | 作用                          | 默认值                                     | 基础配置建议                   |
| :----------- | :-------------------------- | :-------------------------------------- | :----------------------- |
| ssh_args     | 传递给 SSH 命令的额外参数             | 含 ControlMaster=auto ControlPersist=60s | 保持默认，开启 SSH 连接复用         |
| pipelining   | 是否开启 SSH 管道传输               | no                                      | 开启后可大幅减少 SSH 连接次数，提升执行效率 |
| control_path | SSH ControlMaster 套接字文件存储路径 | 系统默认路径                                  | 保持默认即可                   |
    
3. **[privilege_escalation] 段：权限提升专属配置**
    
    集中管理提权相关的所有参数，包括提权密码提示、超时时间、权限验证规则，与 [defaults] 段提权参数联动，可实现精细化的权限控制。
    
4. **[paramiko_connection] 段：Paramiko 库专属配置**
    
    适配不支持 OpenSSH 客户端的系统环境，配置 Paramiko Python SSH 库的连接行为，绝大多数场景保持默认即可。

---

## 本章小结

本章完整覆盖了 Ansible 的核心定位、同类工具技术对比、三大核心特性的底层原理、完整架构与执行流程、标准化安装与基础配置全链路内容，建立了 Ansible 自动化技术的完整理论框架，完成了基础运行环境的搭建。后续章节将基于本章内容，深入讲解 Ansible 资产清单管理、主机连通性配置等核心实操内容。