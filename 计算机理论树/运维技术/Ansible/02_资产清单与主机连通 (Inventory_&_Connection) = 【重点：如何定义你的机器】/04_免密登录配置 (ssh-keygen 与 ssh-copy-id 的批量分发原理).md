SSH 密钥免密登录是 Ansible 管理 Linux/Unix 被控节点的标准认证方式，生产环境严禁使用明文密码认证。本节完整覆盖免密登录的底层原理、标准化配置、批量分发与故障排查。

### 2.5.1 SSH 免密登录核心原理

SSH 密钥认证采用非对称加密算法，分为公钥（Public Key）与私钥（Private Key），核心认证流程如下：

1. 控制节点生成非对称密钥对，私钥严格保存在控制节点本地，权限设置为 600（仅所有者可读），公钥可公开分发；
2. 公钥写入被控节点对应用户的`~/.ssh/authorized_keys`文件，同时配置正确的目录与文件权限；
3. 控制节点发起 SSH 连接时，发送公钥指纹至被控节点，被控节点校验`authorized_keys`中是否存在该公钥；
4. 校验通过后，被控节点生成随机数，用公钥加密后发送至控制节点；
5. 控制节点用本地私钥解密随机数，返回至被控节点，二次校验通过后建立无密码 SSH 连接。

### 2.5.2 标准化免密配置步骤

#### 步骤 1：控制节点生成 SSH 密钥对

推荐使用安全性更高、性能更好的 ed25519 算法，兼容 RSA 算法：

```bash
# 推荐：生成ed25519算法密钥对，注释用于标识密钥用途
ssh-keygen -t ed25519 -C "ansible-control-node"

# 兼容模式：生成RSA 2048位密钥对
ssh-keygen -t rsa -b 2048 -C "ansible-control-node"
```

执行命令后，可自定义密钥存储路径，可选设置密钥密码（Passphrase，生产环境推荐设置，配合`ssh-agent`管理，避免私钥无密码泄露），默认生成路径为`~/.ssh/`目录：

- `id_ed25519`：私钥文件，必须严格保密，权限默认 600；
- `id_ed25519.pub`：公钥文件，可公开分发，无安全风险。

#### 步骤 2：公钥分发至被控节点

使用`ssh-copy-id`工具自动完成公钥写入与权限配置，是官方推荐的标准化分发方式：

```bash
# 基础分发：公钥写入192.168.1.101的ops用户
ssh-copy-id ops@192.168.1.101

# 指定非默认SSH端口
ssh-copy-id -p 2222 ops@192.168.1.101

# 指定自定义公钥文件
ssh-copy-id -i ~/.ssh/id_ed25519.pub ops@192.168.1.101
```

执行后输入被控节点对应用户的 SSH 密码，工具会自动完成：

1. 校验被控节点`~/.ssh`目录是否存在，不存在则自动创建，权限设置为 700；
2. 将公钥追加至`~/.ssh/authorized_keys`文件，权限设置为 600；
3. 修复 SELinux 上下文（若开启），避免认证拦截。

#### 步骤 3：免密登录验证

```bash
# 直接SSH连接，无需输入密码即可登录则配置成功
ssh ops@192.168.1.101
```

### 2.5.3 批量公钥分发方案

针对大规模集群，提供两种标准化批量分发方式，覆盖不同场景：

#### 方式 1：Shell 循环 + sshpass 批量分发（中小规模集群）

`sshpass`工具可自动输入 SSH 密码，无需人工交互，适用于几十台节点的批量操作：

```bash
# 1. 安装sshpass
# RPM系
sudo yum install -y sshpass
# Debian系
sudo apt install -y sshpass

# 2. 批量分发脚本
HOSTS="192.168.1.101 192.168.1.102 192.168.1.103"
USER="ops"
SSH_PASS="your_ssh_password"
for host in $HOSTS; do
  sshpass -p "$SSH_PASS" ssh-copy-id -o StrictHostKeyChecking=no -i ~/.ssh/id_ed25519.pub $USER@$host
done
```

`StrictHostKeyChecking=no` 关闭首次连接的主机密钥确认，适合批量操作，生产环境按需启用。

#### 方式 2：Ansible authorized_key 模块批量分发（已有初始密码认证场景）

使用 Ansible 官方模块批量分发公钥，适用于已完成 Inventory 配置、可通过密码认证连接的场景：

```bash
# 批量分发公钥至all组所有主机，-k参数提示输入SSH连接密码
ansible all -m authorized_key -a "user=ops state=present key={{ lookup('file', '~/.ssh/id_ed25519.pub') }}" -k
```

### 2.5.4 常见故障排查

免密登录失败的 90% 以上场景源于权限配置错误，核心故障点与解决方案如下：

1. **文件权限异常**：控制节点私钥权限必须为 600，被控节点`~/.ssh`目录权限必须为 700，`authorized_keys`文件权限必须为 600，SSH 会拒绝权限过宽的密钥文件；
2. **SSH 配置禁用密钥认证**：检查被控节点`/etc/ssh/sshd_config`中`PubkeyAuthentication yes`，确保密钥认证开启，修改后需重启 sshd 服务；
3. **SELinux 拦截**：被控节点 SELinux 开启时，非默认路径的`.ssh`目录会被拦截，执行`restorecon -R ~/.ssh`修复上下文；
4. **家目录权限异常**：被控节点对应用户的家目录权限不可高于 755，严禁设置为 777，否则 SSH 会拒绝认证；
5. **公钥格式错误**：`authorized_keys`中的公钥必须为单行，不可换行、截断，格式需完整。

---
