命令执行模块是 Ad-Hoc 命令最常用的核心模块，用于在被控节点执行系统命令、脚本，核心包含`command`、`shell`、`script`三个模块，三者的底层逻辑、安全特性、适用场景有严格区分，是必须掌握的核心知识点。

### 3.2.1 command 模块

**核心定义**：Ansible 的默认模块（不指定 - m 参数时自动调用），用于在被控节点执行**非交互式、无 Shell 特性**的系统命令，直接 fork 子进程执行，不通过 /bin/sh Shell 解析，内置安全校验，是官方优先推荐的基础命令执行模块。

#### 核心参数

|参数|核心作用|幂等性支持|
|:--|:--|:--|
|`free_form`|必填，要执行的系统命令（无需显式声明，直接写在 - a 参数中）|否|
|`chdir`|执行命令前，切换到被控节点的指定目录|否|
|`creates`|若指定的文件 / 目录存在，则不执行该命令，用于实现幂等性|是|
|`removes`|若指定的文件 / 目录不存在，则不执行该命令，用于实现幂等性|是|
|`argv`|以列表形式传递命令与参数，避免空格解析风险，安全优先级最高|否|
|`warn`|是否开启危险命令告警（如 rm -rf），默认值为 yes|否|

#### 标准示例

```bash
# 1. 基础执行：查看系统负载
ansible all -m command -a "uptime"

# 2. 切换目录执行命令
ansible all -m command -a "chdir=/opt ls -l"

# 3. 幂等性控制：仅当初始化标记文件不存在时，执行初始化命令
ansible all -m command -a "creates=/data/.initialized sh /opt/init.sh"

# 4. 安全列表传参：避免空格与特殊字符解析异常
ansible all -m command -a "argv=['ls', '-l', '/root/My Documents']"
```

#### 核心特性与注意事项

1. **安全特性**：不通过 Shell 解析，禁止管道符`|`、重定向`> <`、通配符`*`、环境变量`$HOME`、逻辑运算符`&& ||`等 Shell 特性，从根源避免 Shell 注入攻击；
2. **幂等性**：无内置幂等性，必须通过`creates/removes`参数手动实现，否则多次执行会重复运行命令，产生副作用；
3. **使用限制**：无法执行 Shell 内置命令（如 cd、export），cd 操作需通过`chdir`参数实现；
4. **官方最佳实践**：能使用 command 模块实现的场景，优先使用 command，仅当必须使用 Shell 特性时，才切换到 shell 模块。

### 3.2.2 shell 模块

**核心定义**：与 command 模块功能对等，核心区别是**通过被控节点的 /bin/sh Shell 进程执行命令**，完全支持所有 Shell 特性，包括管道、重定向、通配符、环境变量、逻辑运算符、Shell 内置命令，灵活性更高，但存在安全风险。

#### 核心参数

与 command 模块完全一致，支持`chdir`、`creates`、`removes`、`argv`、`warn`等所有参数，用法完全相同。

#### 标准示例

```bash
# 1. 管道符执行：过滤磁盘使用率
ansible all -m shell -a "df -h | grep -v tmpfs | awk '{print \$5,\$6}'"

# 2. 重定向与逻辑运算符：写入文件并校验
ansible all -m shell -a "echo 'net.ipv4.ip_forward = 1' > /etc/sysctl.d/ip_forward.conf && sysctl -p /etc/sysctl.d/ip_forward.conf" -b

# 3. 通配符匹配：批量清理日志文件
ansible all -m shell -a "rm -f /var/log/nginx/*.log.*" -b

# 4. 环境变量调用与条件判断
ansible all -m shell -a "if [ -f /etc/os-release ]; then cat /etc/os-release | grep PRETTY_NAME; fi"
```

#### 核心特性与注意事项

1. **灵活性**：完全支持所有 Shell 语法与特性，可实现复杂的命令组合与逻辑判断；
2. **安全风险**：存在 Shell 注入攻击风险，若命令中包含用户可控的变量，必须严格校验输入，避免恶意代码执行；
3. **幂等性**：无内置幂等性，需通过`creates/removes`参数手动实现；
4. **转义注意事项**：命令中的`$`符号需添加反斜杠转义（`\$`），避免被控制节点的 Shell 提前解析。

### 3.2.3 script 模块

**核心定义**：用于将**控制节点本地的脚本文件**，自动推送到被控节点临时目录并远程执行，无需提前将脚本上传到被控节点，执行完成后自动清理临时文件，是批量执行本地脚本的核心模块。

#### 核心参数

| 参数                | 核心作用                                    | 幂等性支持 |
| :---------------- | :-------------------------------------- | :---- |
| `free_form`       | 必填，控制节点本地的脚本文件路径（相对 / 绝对路径）             | 否     |
| `chdir`           | 执行脚本前，切换到被控节点的指定目录                      | 否     |
| `creates/removes` | 同 command 模块，用于实现幂等性控制                  | 是     |
| `executable`      | 指定执行脚本的解释器，如 /bin/bash、/usr/bin/python3 | 否     |

#### 标准示例

```bash
# 1. 基础执行：运行控制节点本地的Shell脚本
ansible all -m script -a "/opt/ansible/scripts/system_init.sh"

# 2. 指定Python解释器执行本地脚本
ansible all -m script -a "executable=/usr/bin/python3 /opt/ansible/scripts/monitor_check.py"

# 3. 幂等性控制：仅当标记文件不存在时执行脚本
ansible all -m script -a "creates=/data/.init_done /opt/ansible/scripts/init.sh" -b
```

#### 核心特性与注意事项

1. **核心优势**：脚本仅需存放在控制节点本地，无需提前分发到所有被控节点，Ansible 自动完成推送、执行、清理全流程；
2. **执行环境**：脚本在被控节点的系统环境中执行，脚本依赖的命令、工具、依赖包必须在被控节点提前安装；
3. **兼容性**：支持所有类型的脚本（Shell、Python、Perl 等），只需通过`executable`参数指定正确的解释器；
4. **幂等性**：无内置幂等性，需通过`creates/removes`参数控制执行逻辑。

### 3.2.4 三大命令执行模块核心对比表

| 对比维度       | command 模块           | shell 模块                 | script 模块          |
| :--------- | :------------------- | :----------------------- | :----------------- |
| 执行环境       | 被控节点，直接 fork 子进程     | 被控节点，通过 /bin/sh Shell 执行 | 控制节点本地脚本，推送到被控节点执行 |
| Shell 特性支持 | 不支持                  | 完全支持                     | 取决于脚本解释器           |
| 安全风险       | 极低，无 Shell 注入风险      | 高，存在 Shell 注入风险          | 中，取决于脚本内容          |
| 脚本依赖       | 无需脚本，直接执行命令          | 无需脚本，直接执行命令              | 需控制节点本地存在脚本文件      |
| 幂等性        | 需手动实现                | 需手动实现                    | 需手动实现              |
| 官方推荐优先级    | 最高                   | 中等                       | 中等                 |
| 核心适用场景     | 简单、无 Shell 特性的安全命令执行 | 需管道、重定向等 Shell 特性的复杂命令   | 批量执行本地已开发完成的脚本     |

---