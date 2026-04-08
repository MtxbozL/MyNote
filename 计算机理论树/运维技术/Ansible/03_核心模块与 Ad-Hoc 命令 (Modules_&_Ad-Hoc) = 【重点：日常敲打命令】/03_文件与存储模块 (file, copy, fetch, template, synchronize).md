文件与存储模块是 Ansible 管理被控节点文件系统的核心工具，覆盖文件 / 目录创建、文件分发、拉取、动态渲染、增量同步全场景，所有官方模块均**内置完整幂等性**，无需手动处理重复执行的副作用。

### 3.3.1 file 模块

**核心定义**：用于管理被控节点的文件、目录、软链接、硬链接的生命周期与属性，包括创建、删除、权限修改、所有者调整、时间戳更新等，是文件系统操作的基础模块。

#### 核心参数

|参数|核心作用|必填项|
|:--|:--|:--|
|`path`|被控节点的目标文件 / 目录 / 链接的绝对路径|是|
|`state`|核心操作类型，可选值：`file`（默认）、`directory`、`touch`、`link`、`hard`、`absent`|否|
|`src`|源文件路径，仅创建软 / 硬链接时必填|否|
|`mode`|文件 / 目录的权限，如 0755、0644，支持符号格式 u=rwx,g=rx,o=rx|否|
|`owner`|文件 / 目录的所有者用户名 / UID|否|
|`group`|文件 / 目录的所属组名 / GID|否|
|`recurse`|递归修改目录下所有内容的属性，仅 state=directory 时有效，布尔值|否|
|`follow`|是否跟随软链接，修改目标源文件的属性，默认 no|否|

#### state 参数核心取值说明

|取值|核心作用|
|:--|:--|
|`directory`|递归创建目录（等同于 mkdir -p），若目录已存在则仅修改属性|
|`touch`|创建空文件，若文件已存在则更新其访问 / 修改时间戳|
|`link`|创建软链接（符号链接），需配合 src 参数指定源文件|
|`hard`|创建硬链接，需配合 src 参数指定源文件|
|`absent`|递归删除文件、目录、链接（等同于 rm -rf）|
|`file`|默认值，仅检查路径是否存在，修改已有文件 / 目录的属性，不创建新内容|

#### 标准示例

```bash
# 1. 递归创建目录，设置权限与所有者，递归修改子目录属性
ansible all -m file -a "path=/data/app state=directory mode=0755 owner=ops group=ops recurse=yes" -b

# 2. 创建空文件，设置权限
ansible all -m file -a "path=/tmp/app.log state=touch mode=0644 owner=ops group=ops"

# 3. 创建软链接
ansible all -m file -a "src=/data/app/nginx.conf path=/etc/nginx/nginx.conf state=link" -b

# 4. 递归删除目录/文件
ansible all -m file -a "path=/tmp/old_data state=absent"

# 5. 修改已有文件的权限与所有者
ansible all -m file -a "path=/etc/ssh/sshd_config mode=0600 owner=root group=root" -b
```

#### 核心注意事项

1. 完全内置幂等性，仅当系统状态与目标状态不一致时才执行变更，多次执行无副作用；
2. 权限参数`mode`必须添加前导 0（如 0755），否则会被解析为十进制数，导致权限配置错误，是最常见的踩坑点；
3. `state=absent`为递归删除，等同于 rm -rf，生产环境必须先通过`-C`参数干跑验证，避免误删数据。

### 3.3.2 copy 模块

**核心定义**：用于将**控制节点本地的文件 / 目录**复制到被控节点的指定路径，同时支持权限设置、备份、内容校验、配置合法性校验等功能，是静态文件批量分发的核心模块。

#### 核心参数

|参数|核心作用|必填项|
|:--|:--|:--|
|`src`|控制节点本地的源文件 / 目录路径，相对 / 绝对路径，与 content 二选一|否|
|`dest`|被控节点的目标绝对路径|是|
|`content`|直接指定文件内容，替代 src 参数，用于直接生成小文件|否|
|`mode/owner/group`|同 file 模块，设置目标文件的权限、所有者、所属组|否|
|`backup`|覆盖文件前是否备份原文件（带时间戳），默认 no|否|
|`force`|是否强制覆盖已有文件，yes = 总是覆盖，no = 仅当目标不存在时复制，默认 yes|否|
|`validate`|复制完成后执行校验命令，验证文件合法性，% s 会替换为目标文件路径|否|
|`directory_mode`|递归复制目录时，单独设置目录的权限|否|

#### 标准示例

```bash
# 1. 基础分发本地配置文件到被控节点，设置权限
ansible webservers -m copy -a "src=./files/nginx.conf dest=/etc/nginx/nginx.conf mode=0644 owner=root group=root" -b

# 2. 覆盖前备份原文件
ansible webservers -m copy -a "src=./files/nginx.conf dest=/etc/nginx/nginx.conf backup=yes" -b

# 3. 直接通过content生成配置文件，无需本地源文件
ansible all -m copy -a "content='nameserver 223.5.5.5\nnameserver 223.6.6.6\n' dest=/etc/resolv.conf mode=0644" -b

# 4. 复制前校验sudoers文件合法性，校验失败则回滚
ansible all -m copy -a "src=./files/sudoers dest=/etc/sudoers validate='visudo -cf %s'" -b

# 5. 仅当目标文件不存在时才复制，不覆盖已有文件
ansible all -m copy -a "src=./files/init.sh dest=/data/init.sh force=no"
```

#### 核心注意事项

1. 内置幂等性，基于 SHA1 校验和对比源文件与目标文件的内容，仅当内容 / 属性不一致时才执行覆盖；
2. 路径规则：src 路径以`/`结尾时，复制目录内的所有内容到 dest 路径；不以`/`结尾时，将整个目录复制到 dest 路径下；
3. 仅支持控制节点到被控节点的单向复制，不支持跨被控节点的文件传输；
4. 大文件（>100MB）或大量小文件的复制，推荐使用 synchronize 模块，基于 rsync 实现增量传输，性能更优。

### 3.3.3 fetch 模块

**核心定义**：copy 模块的反向操作，用于将**被控节点的文件**批量拉取到控制节点本地，是批量收集日志、配置文件、备份数据的核心模块。

#### 核心参数

|参数|核心作用|必填项|
|:--|:--|:--|
|`src`|被控节点的源文件绝对路径，**仅支持单个文件，不支持目录**|是|
|`dest`|控制节点本地的目标目录路径|是|
|`flat`|是否扁平化存储，默认 no；no = 生成主机名子目录，yes = 直接保存到 dest 目录|否|
|`validate_checksum`|拉取完成后校验文件校验和，确保传输完整，默认 yes|否|
|`fail_on_missing`|源文件不存在时是否执行失败，默认 yes；no = 忽略不存在的文件|否|

#### 标准示例

```bash
# 1. 批量拉取所有被控节点的sshd_config到控制节点，默认按主机名分目录存储
ansible all -m fetch -a "src=/etc/ssh/sshd_config dest=./backup/sshd_config/"

# 2. 扁平化拉取主机名文件，直接保存到目标目录
ansible all -m fetch -a "src=/etc/hostname dest=./hostnames/ flat=yes"

# 3. 拉取应用日志，忽略不存在的文件
ansible webservers -m fetch -a "src=/var/log/nginx/access.log dest=./logs/access/ fail_on_missing=no"
```

#### 核心注意事项

1. 内置幂等性，仅当控制节点本地文件与被控节点源文件校验和不一致时才执行拉取；
2. 仅支持拉取单个文件，不支持直接拉取目录，如需拉取目录，需先在被控节点打包，再拉取压缩包；
3. 默认非扁平化存储，会在 dest 目录下生成`主机名/原文件路径`的层级结构，避免多台主机的同名文件互相覆盖，生产环境推荐使用默认模式；
4. `flat=yes`模式下，多台主机的同名文件会互相覆盖，仅当文件名唯一时使用。

### 3.3.4 template 模块

**核心定义**：基于 Jinja2 模板引擎，将控制节点本地的 Jinja2 模板文件渲染后，复制到被控节点的指定路径，是动态生成配置文件的核心模块，是变量与模板体系的前置基础。

#### 核心参数

核心参数与 copy 模块完全一致，支持`src`、`dest`、`mode`、`owner`、`backup`、`validate`等所有参数，额外支持 Jinja2 模板的分隔符自定义参数。

#### 标准示例

```bash
# 基础模板渲染：将Jinja2模板渲染后分发到被控节点
ansible webservers -m template -a "src=./templates/nginx.conf.j2 dest=/etc/nginx/nginx.conf mode=0644" -b
```

#### 核心特性与注意事项

1. 内置幂等性，基于渲染后的文件内容校验和判断是否需要覆盖，多次执行无副作用；
2. 模板渲染在**控制节点本地完成**，被控节点无需安装 Jinja2 依赖；
3. 与 copy 模块的核心区别：copy 模块是静态文件复制，template 模块支持 Jinja2 变量替换、条件判断、循环等动态逻辑，是配置文件动态生成的官方标准方案；
4. 完整的 Jinja2 语法体系将在第五章详细讲解。

### 3.3.5 synchronize 模块

**核心定义**：基于 rsync 工具实现的增量文件同步模块，支持控制节点与被控节点之间的双向同步，性能优异，适合大文件、大量小文件、目录的批量同步场景。

#### 核心参数

|参数|核心作用|必填项|
|:--|:--|:--|
|`src`|源路径，支持控制节点 / 被控节点路径|是|
|`dest`|目标路径，支持控制节点 / 被控节点路径|是|
|`mode`|同步模式，push（默认，控制节点推送到被控节点）、pull（被控节点拉取到控制节点）|否|
|`archive`|归档模式，等同于 rsync -a，开启递归、保留权限 / 所有者 / 时间戳 / 软链接，默认 yes|否|
|`delete`|删除目标路径中源路径不存在的文件，实现完全镜像同步，默认 no|否|
|`compress`|传输时开启压缩，提升慢网络环境的传输效率，默认 yes|否|
|`checksum`|基于校验和判断文件变化，而非时间戳与大小，默认 no|否|
|`rsync_opts`|传递给 rsync 的额外参数，如 --exclude、--include|否|

#### 标准示例

```bash
# 1. 控制节点代码目录增量推送到被控节点
ansible webservers -m synchronize -a "src=./app_code/ dest=/data/app/"

# 2. 批量拉取被控节点的Nginx日志目录到控制节点
ansible webservers -m synchronize -a "src=/var/log/nginx/ dest=./logs/nginx/ mode=pull"

# 3. 完全镜像同步，删除目标路径多余文件
ansible webservers -m synchronize -a "src=./app_code/ dest=/data/app/ delete=yes"

# 4. 排除日志文件与.git目录同步
ansible webservers -m synchronize -a "src=./app_code/ dest=/data/app/ rsync_opts=--exclude=*.log --exclude=.git/"
```

#### 核心注意事项

1. 前置依赖：**控制节点与被控节点都必须安装 rsync 工具**，否则模块执行失败；
2. 内置幂等性，基于 rsync 增量算法，仅同步发生变化的文件块，传输性能远优于 copy/fetch 模块；
3. 支持双向同步，push 模式等同于 copy，pull 模式等同于 fetch，灵活性更强；
4. `delete=yes`会删除目标路径中源路径不存在的文件，实现完全镜像同步，生产环境必须先通过`-C`参数干跑验证，避免误删数据。

---
