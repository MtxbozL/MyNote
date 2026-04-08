
系统与用户模块是 Ansible 管理被控节点系统配置的核心工具，覆盖用户 / 组管理、定时任务、主机名、内核参数等核心系统操作，所有模块均**内置完整幂等性**。

### 3.5.1 user 模块

**核心定义**：用于管理被控节点的系统用户，包括创建、修改、删除用户，设置密码、UID、家目录、Shell、用户组、过期时间等全生命周期管理。

#### 核心参数

|参数|核心作用|必填项|
|:--|:--|:--|
|`name`|用户名|是|
|`state`|操作状态，present（默认）、absent|否|
|`uid`|指定用户的 UID|否|
|`gid`|指定用户的主 GID|否|
|`groups`|用户的附属组列表，多个用逗号分隔|否|
|`append`|是否追加附属组，默认 no（覆盖），yes（追加不删除现有组）|否|
|`home`|用户的家目录路径|否|
|`create_home`|是否创建家目录，默认 yes|否|
|`shell`|用户的登录 Shell，如 /bin/bash、/sbin/nologin|否|
|`password`|用户的加密密码，必须是 /etc/shadow 格式的加密字符串|否|
|`expires`|用户过期时间，Unix 时间戳格式|否|
|`system`|是否创建系统用户，默认 no|否|
|`remove`|删除用户时是否同时删除家目录 / 邮箱，state=absent 时有效，默认 no|否|

#### 标准示例

```bash
# 1. 创建普通运维用户，设置Shell与家目录
ansible all -m user -a "name=ops state=present shell=/bin/bash home=/home/ops create_home=yes" -b

# 2. 创建系统用户，禁止登录
ansible all -m user -a "name=nginx state=present system=yes shell=/sbin/nologin" -b

# 3. 给用户追加docker与wheel附属组，不覆盖现有组
ansible all -m user -a "name=ops groups=docker,wheel append=yes" -b

# 4. 删除用户，同时删除家目录与邮箱
ansible all -m user -a "name=test state=absent remove=yes" -b
```

#### 核心注意事项

1. 必须提权执行，内置幂等性；
2. `password`参数必须是加密后的字符串，明文密码会直接写入 /etc/shadow，导致无法登录，加密密码可通过`mkpasswd`命令或 Python crypt 模块生成；
3. `append=yes`是高频踩坑点，默认 no 会覆盖用户的现有附属组，导致权限丢失，追加附属组时必须设置`append=yes`；
4. `remove=yes`会彻底删除用户家目录，谨慎使用。

### 3.5.2 group 模块

**核心定义**：用于管理被控节点的系统用户组，包括创建、修改、删除组，设置 GID、组类型等。

#### 核心参数与标准示例

核心参数包括`name`（组名，必填）、`state`（present/absent）、`gid`（指定 GID）、`system`（是否系统组）。

```bash
# 1. 创建普通用户组
ansible all -m group -a "name=devops state=present" -b

# 2. 创建系统组，指定GID
ansible all -m group -a "name=docker state=present system=yes gid=999" -b

# 3. 删除用户组
ansible all -m group -a "name=test state=absent" -b
```

#### 核心注意事项

必须提权执行，内置幂等性；删除组时，若该组是某个用户的主组，会执行失败，需先修改用户的主组。

### 3.5.3 cron 模块

**核心定义**：用于管理被控节点的 crontab 定时任务，包括创建、修改、删除、禁用任务，支持用户级与系统级 crontab，不会修改非 Ansible 管理的定时任务。

#### 核心参数

|参数|核心作用|必填项|
|:--|:--|:--|
|`name`|定时任务的唯一名称，用于 Ansible 识别任务，避免重复创建|是|
|`job`|定时任务执行的命令 / 脚本，必须使用绝对路径|是|
|`user`|crontab 所属的用户，默认 root|否|
|`state`|操作状态，present（默认）、absent|否|
|`disabled`|是否禁用任务，默认 no，yes = 注释任务，保留条目|否|
|`minute/hour/day/month/weekday`|对应 crontab 的分、时、日、月、周，默认值均为 *|否|
|`special_time`|特殊时间预设，替代分时分月周，如 @reboot、@daily、@hourly|否|
|`cron_file`|指定系统级 crontab 文件路径，通常位于 /etc/cron.d/|否|

#### 标准示例

```bash
# 1. 创建每天凌晨2点执行的备份任务
ansible all -m cron -a "name='daily_backup' job='/opt/scripts/backup.sh' minute=0 hour=2" -b

# 2. 创建每5分钟执行一次的监控任务
ansible all -m cron -a "name='monitor_check' job='/opt/scripts/monitor.sh' minute=*/5" -b

# 3. 创建每周一到周五早上8点执行的任务
ansible all -m cron -a "name='workday_notify' job='/opt/scripts/notify.sh' minute=0 hour=8 weekday=1-5" -b

# 4. 开机自启执行的初始化任务
ansible all -m cron -a "name='startup_init' job='/opt/scripts/init.sh' special_time=@reboot" -b

# 5. 禁用已有的定时任务
ansible all -m cron -a "name='daily_backup' disabled=yes job='/opt/scripts/backup.sh' minute=0 hour=2" -b

# 6. 删除定时任务
ansible all -m cron -a "name='daily_backup' state=absent" -b
```

#### 核心注意事项

1. 必须提权执行，内置幂等性，基于`name`参数唯一标识任务，不会重复创建或修改其他任务；
2. `name`参数为必填项，且必须唯一，否则会导致任务被覆盖；
3. `job`参数必须使用绝对路径，crontab 的执行环境 PATH 变量有限，相对路径会导致命令执行失败；
4. `disabled=yes`时，必须同时提供`job`与原时间参数，否则会导致任务内容被清空。

### 3.5.4 hostname 模块

**核心定义**：用于修改被控节点的系统主机名，同时更新 /etc/hostname 文件与内核主机名，适配 systemd 与非 systemd 系统。

#### 标准示例

```bash
# 修改主机名为web01.example.com
ansible 192.168.1.101 -m hostname -a "name=web01.example.com" -b
```

#### 核心注意事项

必须提权执行，内置幂等性；修改主机名后，不会自动更新 /etc/hosts 文件，需配合 lineinfile 模块同步修改。

### 3.5.5 sysctl 模块

**核心定义**：用于管理被控节点的内核参数，修改 /proc/sys/ 下的内核配置，支持临时生效与永久写入配置文件，自动执行 sysctl 重载。

#### 核心参数

|参数|核心作用|必填项|
|:--|:--|:--|
|`name`|内核参数名称，如 net.ipv4.ip_forward|是|
|`value`|内核参数的目标值|是|
|`state`|操作状态，present（默认）、absent|否|
|`sysctl_file`|指定写入的配置文件，默认 /etc/sysctl.conf|否|
|`reload`|修改后是否执行 sysctl -p 重载，默认 yes|否|
|`sysctl_set`|是否立即设置内核参数，默认 yes|否|

#### 标准示例

```bash
# 1. 开启IPv4转发，永久生效
ansible all -m sysctl -a "name=net.ipv4.ip_forward value=1 state=present" -b

# 2. 优化TCP参数，写入专属配置文件
ansible all -m sysctl -a "name=net.ipv4.tcp_tw_reuse value=1 sysctl_file=/etc/sysctl.d/99-tcp.conf" -b

# 3. 禁用IPv6
ansible all -m sysctl -a "name=net.ipv6.conf.all.disable_ipv6 value=1" -b
```

#### 核心注意事项

必须提权执行，内置幂等性；生产环境推荐将自定义内核参数写入 /etc/sysctl.d/ 下的专属文件，而非直接修改 /etc/sysctl.conf，便于管理。

---
