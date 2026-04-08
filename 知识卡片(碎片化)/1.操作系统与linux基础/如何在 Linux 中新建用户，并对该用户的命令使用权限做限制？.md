#### 核心标准答案

Linux 中新建用户和权限限制，核心是通过**用户管理命令、sudo 权限控制、受限 shell、文件系统权限、ACL 访问控制**等方式实现，完整操作流程如下：

##### 一、新建用户的核心操作

1. **新建用户基础命令**
    
    使用`useradd`命令新建用户，`passwd`命令设置用户密码，核心命令：
    
    bash
    
    运行
    
    ```
    # 新建用户，自动创建家目录、默认shell为bash
    useradd -m testuser
    # 为用户设置密码，交互输入密码
    passwd testuser
    ```
    
    常用参数：
    
    - `-m`/`--create-home`：自动创建用户家目录`/home/testuser`；
    - `-s`/`--shell`：指定用户的默认 shell，比如`-s /bin/bash`、`-s /sbin/nologin`；
    - `-d`/`--home-dir`：指定用户的家目录；
    - `-g`/`--gid`：指定用户的主组；
    - `-G`/`--groups`：指定用户的附属组。
    
2. **用户基础验证**
    
    执行`id testuser`，查看用户的 UID、GID、所属组，确认用户创建成功；执行`su - testuser`，切换到该用户，验证登录正常。
    

##### 二、对用户的命令使用权限做限制

核心有 5 种常用方式，从简单到严格依次如下：

###### 1. 方式 1：通过 sudoers 文件，精细化控制用户的 sudo 命令权限

这是最常用的方式，控制用户可以通过 sudo 执行哪些命令，禁止执行其他 root 权限命令。

- 核心操作：
    
    1. 执行`visudo`命令，编辑`/etc/sudoers`文件（必须用 visudo，会自动校验语法，避免语法错误导致 sudo 失效）；
    2. 在文件末尾添加配置，精细化控制命令权限：
    
    bash
    
    运行
    
    ```
    # 允许testuser执行systemctl restart nginx、df、free命令，禁止执行其他sudo命令
    testuser ALL=(ALL) /usr/bin/systemctl restart nginx, /usr/bin/df, /usr/bin/free
    # 允许testuser执行所有命令，禁止执行passwd root、rm -rf /等危险命令
    testuser ALL=(ALL) ALL, !/usr/bin/passwd root, !/usr/bin/rm -rf /
    # 允许testuser执行所有命令，无需输入密码
    testuser ALL=(ALL) NOPASSWD: ALL
    ```
    
    3. 保存退出，visudo 会自动校验语法，语法正确则生效；
    4. 验证：切换到 testuser，执行`sudo 命令`，验证权限是否符合预期。
    

###### 2. 方式 2：使用受限 shell（rbash），限制用户的命令执行范围

受限 shell 会禁止用户执行 cd、修改环境变量、执行绝对路径外的命令、重定向等操作，大幅限制用户的操作权限，适合只允许用户执行少量固定命令的场景。

- 核心操作：
    
    1. 创建用户时，指定默认 shell 为 rbash：
    
    bash
    
    运行
    
    ```
    useradd -m -s /bin/rbash testuser
    passwd testuser
    ```
    
    2. 限制用户的 PATH 环境变量，只允许执行指定目录下的命令：
    
    bash
    
    运行
    
    ```
    # 切换到root，创建用户的命令白名单目录
    mkdir /home/testuser/bin
    # 只允许用户执行ls、df、free命令，创建软链接到白名单目录
    ln -s /usr/bin/ls /home/testuser/bin/
    ln -s /usr/bin/df /home/testuser/bin/
    ln -s /usr/bin/free /home/testuser/bin/
    # 锁定用户的PATH环境变量，禁止修改
    echo "export PATH=/home/testuser/bin" >> /home/testuser/.bashrc
    # 锁定用户的配置文件，禁止修改
    chattr +i /home/testuser/.bashrc
    chown root:root /home/testuser/.bashrc
    chmod 644 /home/testuser/.bashrc
    ```
    
    3. 验证：切换到 testuser，只能执行白名单内的 ls、df、free 命令，其他命令都会提示找不到，同时禁止 cd 到其他目录、修改环境变量等操作。
    

###### 3. 方式 3：通过文件系统权限，限制用户对文件 / 目录的访问

Linux 的文件权限分为所有者、所属组、其他用户，通过修改文件 / 目录的权限，限制用户的读、写、执行权限。

- 核心操作：
    
    bash
    
    运行
    
    ```
    # 禁止其他用户访问/root目录，testuser属于其他用户，无法访问
    chmod 700 /root
    # 禁止testuser修改/etc目录下的配置文件，默认/etc目录权限为755，只有root可写
    chmod 755 /etc
    # 给testuser开放某个目录的读写权限，其他目录只读
    chown -R testuser:testuser /data/testuser
    chmod 700 /data/testuser
    ```
    

###### 4. 方式 4：通过 ACL 访问控制列表，精细化控制单个用户的权限

ACL 是 Linux 文件权限的扩展，可针对单个用户、单个组设置精细化的权限，比传统的 ugo 权限更灵活。

- 核心操作：
    
    bash
    
    运行
    
    ```
    # 安装ACL工具
    yum install acl / apt install acl
    # 禁止testuser访问/var/log目录
    setfacl -m u:testuser:--- /var/log
    # 允许testuser对/data目录有读和执行权限，无写权限
    setfacl -m u:testuser:r-x /data
    # 查看ACL权限
    getfacl /var/log
    # 删除ACL权限
    setfacl -x u:testuser /var/log
    ```
    

###### 5. 方式 5：chroot 监狱，限制用户只能在指定的目录内操作

chroot 会将用户的根目录切换到指定的目录，用户无法访问目录外的任何文件，实现完全的环境隔离，是最严格的权限限制方式。

#### 专家级拓展

- 生产环境最佳实践：
    
    1. 普通业务用户，优先使用 sudoers 文件控制权限，遵循最小权限原则，只给用户分配必要的命令权限，禁止直接使用 root 用户；
    2. 堡垒机 / 跳板机用户，优先使用 rbash 受限 shell + 命令白名单，严格限制用户的操作范围，同时开启操作审计，记录用户的所有命令；
    3. 权限配置完成后，必须做全面的验证，确认权限限制符合预期，没有越权操作的漏洞；
    4. 禁止给普通用户配置 NOPASSWD: ALL 的 sudo 权限，会导致用户可以直接提权到 root，存在严重的安全风险；
    5. 定期审计用户的权限配置，清理无用的用户和权限，避免权限泄露。
    
- 进阶安全配置：
    
    1. 禁用用户的 SSH 密码登录，只允许密钥登录，提升登录安全性；
    2. 配置 PAM 模块，限制用户的登录时间、登录 IP、同时登录的终端数；
    3. 开启 Linux 的审计功能（auditd），记录用户的所有操作，方便事后审计和故障排查。
    

#### 面试避坑指南

- 严禁只说新建用户的命令，不说权限限制的具体方式，面试问这个问题，核心是考察你对 Linux 权限控制的理解，而不只是 useradd 命令；
- 避免编辑 sudoers 文件时直接用 vi/vim，必须用 visudo 命令，否则语法错误会导致所有 sudo 命令失效，这是生产环境的核心坑点，提到会大幅加分；
- 不要只说一种权限限制方式，要根据场景说明不同方式的适用范围，体现你对 Linux 权限体系的全面理解。