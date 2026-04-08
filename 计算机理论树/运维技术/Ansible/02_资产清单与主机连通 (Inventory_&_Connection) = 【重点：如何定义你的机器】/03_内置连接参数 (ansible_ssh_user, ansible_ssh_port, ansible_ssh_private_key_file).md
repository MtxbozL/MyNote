Ansible 提供了标准化的内置主机变量，用于定义单主机 / 组的远程连接行为、认证方式、权限提升、解释器路径等核心配置，是保障被控节点连通性的核心。参数可定义在主机级（优先级高）或组级（组内继承），优先级高于`ansible.cfg`全局配置。

### 2.4.1 Linux/Unix 核心 SSH 连接参数

为 Linux/Unix 被控节点的默认连接参数，覆盖 SSH 连接全生命周期配置：

| 参数名                          | 核心作用                                                                     | 配置示例                                                                              |
| :--------------------------- | :----------------------------------------------------------------------- | :-------------------------------------------------------------------------------- |
| ansible_host                 | 目标主机的实际连接地址（IP/FQDN），用于主机别名与实际地址的映射                                      | web01 ansible_host=192.168.1.101                                                  |
| ansible_connection           | 连接插件类型，默认`ssh`，可选`paramiko`（Python SSH 库）、`local`（本地执行）、`winrm`（Windows） | ansible_connection=ssh                                                            |
| ansible_port                 | 远程连接端口，SSH 默认 22                                                         | ansible_port=2222                                                                 |
| ansible_user                 | 远程连接的系统用户名                                                               | ansible_user=ops                                                                  |
| ansible_ssh_private_key_file | SSH 私钥文件的绝对路径，用于密钥认证                                                     | ansible_ssh_private_key_file=/home/ops/.ssh/id_ed25519                            |
| ansible_ssh_pass             | SSH 密码认证的明文密码，**生产环境严禁使用**，仅用于测试，需配合 Ansible Vault 加密                    | ansible_ssh_pass=test_pass                                                        |
| ansible_ssh_common_args      | 传递给 SSH/SCP/SFTP 会话的全局额外参数，用于堡垒机 / 跳板机代理配置                               | ansible_ssh_common_args="-o ProxyCommand='ssh -W %h:%p jump@192.168.1.250 -p 22'" |
| ansible_python_interpreter   | 被控节点 Python 解释器的绝对路径，解决多 Python 版本兼容问题                                   | ansible_python_interpreter=/usr/bin/python3.11                                    |

### 2.4.2 权限提升（Become）内置参数

对应第一章的提权机制，用于定义主机 / 组级的权限提升规则，覆盖全场景提权配置：

|参数名|核心作用|配置示例|
|:--|:--|:--|
|ansible_become|是否开启权限提升，布尔值 yes/no|ansible_become=yes|
|ansible_become_method|提权方式，默认`sudo`，可选`su`、`doas`、`pbrun`等|ansible_become_method=sudo|
|ansible_become_user|提权至的目标用户，默认 root|ansible_become_user=root|
|ansible_become_pass|提权密码，**明文严禁生产使用**，需用 Vault 加密|ansible_become_pass=sudo_pass|

### 2.4.3 Windows 被控节点 WinRM 核心参数

用于 Windows 被控节点的远程连接配置，是 Ansible 管理 Windows 主机的核心：

|参数名|核心作用|配置示例|
|:--|:--|:--|
|ansible_connection|必须设置为`winrm`，启用 WinRM 连接插件|ansible_connection=winrm|
|ansible_host|Windows 主机的 IP/FQDN|ansible_host=192.168.1.150|
|ansible_port|WinRM 端口，HTTP 默认 5985，HTTPS 默认 5986|ansible_port=5986|
|ansible_user|Windows 远程管理用户名（本地管理员 / 域用户）|ansible_user=administrator|
|ansible_password|Windows 用户登录密码，需用 Vault 加密|ansible_password=windows_pass|
|ansible_winrm_transport|WinRM 认证方式，推荐`ntlm`/`kerberos`|ansible_winrm_transport=ntlm|
|ansible_winrm_server_cert_validation|SSL 证书验证，自签名证书需设置为`ignore`|ansible_winrm_server_cert_validation=ignore|

---
