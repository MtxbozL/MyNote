## 一、文档概述

本文档为Linux `hostname` 命令的标准化学习与参考文档，完全适配之前的Linux命令笔记体系，属于**01-基础系统与终端操作类**。

- 适用环境：主流Linux发行版（Ubuntu 20.04+/Debian 10+/Rocky Linux 8+/CentOS 7+），兼容SysVinit与systemd系统

- 核心受众：Linux入门学习者、运维工程师、网络配置人员

- 文档价值：覆盖主机名查看、临时/永久修改、域名解析关联、集群节点配置等全场景，解决主机名配置异常、网络访问失败、主机标识混乱、sudo命令卡顿等高频问题。

## 二、核心定位

`hostname` 是Linux系统标配的基础命令，核心作用是**查看、临时修改系统的主机名（Hostname），也可查询系统的域名、FQDN（完全限定域名）、IP绑定关系**。

主机名是Linux系统在网络中的唯一标识，是服务器集群管理、局域网互访、服务配置、日志审计、SSL证书校验的核心基础，是Linux网络配置与系统管理的必备入门命令。

## 三、语法格式

```Bash

# 1. 基础查看语法（无权限要求，普通用户可直接执行）
hostname [可选参数]

# 2. 临时设置主机名语法（需root/sudo权限，重启系统后失效）
sudo hostname [新主机名]

# 3. 从文件读取主机名并设置（需root权限，自动化场景专用）
sudo hostname -F [主机名配置文件路径]
```

核心红线提示：直接用`hostname 新主机名`设置的主机名**仅临时生效**，仅对当前系统运行周期有效，服务器重启后会被配置文件中的默认值覆盖；永久修改需配合配置文件或`hostnamectl`命令，详见下文实操示例。

## 四、高频核心参数

|参数|核心作用|高频使用场景|
|---|---|---|
|无参数（直接执行`hostname`）|输出系统当前的**短主机名（Short Hostname）**|快速查看系统主机标识，日常最基础、最高频用法|
|`-s, --short`|输出短主机名（截取FQDN第一个`.`之前的内容）|从长域名中提取纯主机名，Shell脚本编写、日志标识高频使用|
|`-f, --fqdn, --long`|输出系统的**完全限定域名FQDN**（短主机名+域名后缀）|服务器集群、邮件服务、SSL证书配置、跨网互访场景，必须使用合规FQDN|
|`-d, --domain`|输出系统的DNS域名后缀（Domain Name）|查看域名配置，配合FQDN校验、DNS解析排查|
|`-i, --ip-address`|输出主机名对应的IP地址（解析`/etc/hosts`绑定关系）|快速验证主机名与IP的绑定是否生效，排查解析异常|
|`-I, --all-ip-addresses`|输出主机所有网卡的IP地址（排除回环地址`127.0.0.1`）|无需解析hosts，一键查看本机所有生效IP，比`ip addr`更简洁，运维排查高频|
|`-A, --all-fqdns`|输出系统所有网卡对应的全限定域名|多网卡、多域名的服务器场景，查看全量FQDN配置|
|`-a, --alias`|输出主机的别名（Alias）|查看`/etc/hosts`中配置的主机别名，局域网简化访问场景|
|`-y, --yp, --nis`|输出NIS/YP局域网域名|企业内网NIS统一认证服务配置场景|
|`-F, --file`|从指定文件中读取主机名并设置|自动化部署、脚本批量配置主机名场景|
## 五、实操示例（超丰富全场景覆盖）

### 1. 基础入门示例（新手必练，零门槛上手）

```Bash

# 示例1：最基础用法，查看当前系统的主机名
hostname
# 执行效果：直接输出短主机名，例：ubuntu-server、rocky-node01
```

```Bash

# 示例2：查看系统的完全限定域名FQDN
hostname -f
# 执行效果：输出带域名后缀的完整标识，例：node01.cluster.local
```

```Bash

# 示例3：查看主机名绑定的IP地址（解析/etc/hosts）
hostname -i
# 执行效果：输出/etc/hosts中与当前主机名绑定的IP，例：192.168.1.100
```

```Bash

# 示例4：一键查看本机所有网卡的IP地址（无需过滤，直接获取）
hostname -I
# 执行效果：输出所有生效的网卡IP，例：192.168.1.100 10.0.0.1 172.17.0.1
```

```Bash

# 示例5：临时修改主机名（需sudo权限，重启失效）
sudo hostname test-node01
# 执行效果：立即修改当前系统的主机名，重新打开终端即可生效，重启后恢复
```

---

### 2. 高频日常场景示例（90%运维工作都会用到）

#### 场景1：永久修改主机名（全系统兼容方案+systemd推荐方案）

主机名永久修改必须完成「核心配置修改+hosts同步」两步，否则会出现sudo卡顿、解析失败等问题，以下是2种通用方案：

##### 方案1：全发行版兼容方案（修改配置文件，适配所有Linux系统）

```Bash

# 步骤1：修改主机名核心配置文件
# Debian/Ubuntu系列
sudo vim /etc/hostname
# Rocky/CentOS/RHEL系列（CentOS 6及以前需修改/etc/sysconfig/network）
sudo vim /etc/hostname

# 步骤2：在文件中写入你的新主机名（仅一行，无其他内容），例：prod-node01
# 保存退出vim

# 步骤3：同步修改/etc/hosts文件，避免解析异常（必做！新手最高频踩坑点）
sudo vim /etc/hosts
# 在文件中添加一行，将新主机名绑定到你的服务器IP，例：
127.0.0.1   localhost
192.168.1.100  prod-node01  prod-node01.cluster.local
# 保存退出vim

# 步骤4：使配置立即生效（无需重启）
sudo hostname -F /etc/hostname
# 或执行以下命令强制加载配置
sudo systemctl restart systemd-hostnamed

# 步骤5：验证修改是否生效
hostname
# 重新打开终端，命令行前缀会自动更新为新主机名
```

##### 方案2：systemd系统推荐方案（一行命令永久生效，Ubuntu 16.04+/CentOS 7+通用）

现代Linux发行版均使用systemd，推荐用`hostnamectl`命令一键永久修改，自动同步核心配置：

```Bash

# 一行命令永久修改静态主机名，重启不失效
sudo hostnamectl set-hostname prod-node01

# 同步修改/etc/hosts（仍需手动完成，避免sudo卡顿）
sudo vim /etc/hosts
# 添加IP与主机名的绑定关系，保存退出

# 验证永久配置是否生效
hostnamectl status  # 查看完整的主机名配置
hostname             # 验证短主机名
```

#### 场景2：脚本中动态获取主机名，实现个性化标识

主机名是服务器的唯一标识，在Shell脚本中可动态获取，用于生成带主机标识的备份文件、日志、告警信息，是自动化脚本的高频用法：

```Bash

#!/bin/bash
# 动态获取当前主机名
SERVER_HOST=$(hostname)
BACKUP_DATE=$(date +%F)

# 生成带主机名的备份文件名，集群中可区分不同节点的备份
BACKUP_FILE="backup_${SERVER_HOST}_${BACKUP_DATE}.tar.gz"

# 执行备份逻辑
echo "正在为节点【${SERVER_HOST}】执行备份，备份文件：${BACKUP_FILE}"
tar -zcvf ${BACKUP_FILE} /data/
echo -e "\e[1;32m 节点${SERVER_HOST}备份完成！\e[0m"

# 日志中写入主机标识，方便集群日志聚合排查
echo "[$(date +%F %T)] [${SERVER_HOST}] 备份任务执行成功" >> /var/log/backup.log
```

#### 场景3：主机名与IP绑定校验，排查解析异常

当出现ssh卡顿、sudo命令慢、服务无法绑定主机名等问题时，可通过以下命令快速排查主机名解析是否正常：

```Bash

# 1. 验证主机名是否能正常解析为IP
hostname -i
# 若输出127.0.0.1或无输出，说明/etc/hosts中未正确绑定主机名与IP

# 2. 验证本机所有IP是否正常
hostname -I
# 确认服务器的业务IP是否正常获取

# 3. 验证FQDN配置是否合规
hostname -f
# 若报错"hostname: Name or service not known"，说明FQDN配置错误，需检查/etc/hosts
```

#### 场景4：从文件批量读取主机名，自动化部署

在批量装机、集群节点初始化场景中，可通过`-F`参数从文件读取主机名，实现自动化配置：

```Bash

# 1. 生成主机名配置文件
echo "k8s-worker-03" > /tmp/hostname.conf

# 2. 从文件读取并设置主机名
sudo hostname -F /tmp/hostname.conf

# 3. 同步到永久配置文件
sudo cp /tmp/hostname.conf /etc/hostname

# 4. 验证生效
hostname
```

---

### 3. 进阶奇妙用法（实用骚操作，解锁hostname的隐藏能力）

#### 奇妙用法1：一键修改主机名并同步hosts，永久生效（一行命令搞定）

无需分步编辑多个文件，一行命令完成「永久设置主机名+hosts同步+生效验证」，适合批量操作：

```Bash

# 定义变量：新主机名、服务器IP
NEW_HOST="k8s-master-01"
SERVER_IP="192.168.1.200"

# 一行命令完成全流程
sudo hostnamectl set-hostname $NEW_HOST && \
sudo sed -i "/$NEW_HOST/d" /etc/hosts && \
sudo echo "$SERVER_IP $NEW_HOST ${NEW_HOST}.cluster.local" >> /etc/hosts && \
echo "主机名修改完成，当前主机名：$(hostname)"
```

#### 奇妙用法2：主机名合法性自动校验脚本

主机名必须符合RFC 1123规范，否则会导致网络解析异常，以下脚本可自动校验主机名是否合规：

```Bash

#!/bin/bash
# 主机名合法性校验脚本
CHECK_HOST=$1

# RFC规范：仅允许小写字母、数字、连字符-、点.，不能以连字符开头/结尾，长度≤63
if [[ $# -ne 1 ]]; then
    echo "用法：$0 待校验的主机名"
    exit 2
fi

if [[ $CHECK_HOST =~ ^[a-z0-9][a-z0-9.-]{0,61}[a-z0-9]$ ]]; then
    echo -e "\e[1;32m [SUCCESS] 主机名 $CHECK_HOST 符合RFC规范\e[0m"
    exit 0
else
    echo -e "\e[1;31m [ERROR] 主机名 $CHECK_HOST 不符合规范！\e[0m"
    echo "规范要求：仅允许小写字母、数字、连字符-、点.，不能以连字符开头/结尾，长度不超过63个字符"
    exit 1
fi
```

#### 奇妙用法3：云服务器主机名永久生效，解决cloud-init重置问题

阿里云、腾讯云、华为云等云服务器，默认开启cloud-init服务，重启后会自动重置主机名为云平台默认值，需修改cloud-init配置才能永久生效：

```Bash

# 步骤1：修改cloud-init配置文件，禁用主机名重置
sudo vim /etc/cloud/cloud.cfg

# 步骤2：找到preserve_hostname配置，修改为true（没有则新增一行）
preserve_hostname: true

# 步骤3：用hostnamectl永久设置主机名
sudo hostnamectl set-hostname cloud-prod-node01

# 步骤4：同步/etc/hosts配置
sudo sed -i "/cloud-prod-node01/d" /etc/hosts
sudo echo "127.0.0.1 cloud-prod-node01" >> /etc/hosts

# 步骤5：重启验证，主机名不会再被重置
sudo reboot
```

#### 奇妙用法4：临时修改主机名，退出终端自动恢复（测试场景专用）

测试场景中需要临时修改主机名，又不想影响系统永久配置，可通过子Shell实现，退出终端自动恢复：

```Bash

# 进入一个子Shell，临时修改主机名
sudo bash -c "hostname test-temp-node; exec bash"

# 此时主机名已修改为test-temp-node，执行exit退出子Shell，自动恢复原主机名
exit
```

#### 奇妙用法5：集群节点批量修改主机名的Ansible剧本模板

大规模服务器集群场景中，可通过Ansible批量修改所有节点的主机名，无需逐台登录：

```YAML

# 文件名：set_hostname.yml
- name: 批量修改集群节点主机名
  hosts: all
  become: yes
  tasks:
    - name: 永久设置主机名（使用inventory中的主机名）
      hostnamectl:
        name: "{{ inventory_hostname }}"
        use: systemd

    - name: 同步更新/etc/hosts文件
      lineinfile:
        path: /etc/hosts
        line: "{{ ansible_default_ipv4.address }} {{ inventory_hostname }} {{ inventory_hostname }}.cluster.local"
        state: present
        regexp: "^{{ ansible_default_ipv4.address }}"

    - name: 验证主机名配置
      command: hostname
      register: hostname_result

    - name: 输出修改结果
      debug:
        msg: "节点 {{ inventory_hostname }} 主机名修改完成，当前主机名：{{ hostname_result.stdout }}"
```

执行命令：`ansible-playbook -i inventory.ini set_hostname.yml`

## 六、避坑提示&注意事项

1. **临时修改vs永久修改的核心坑（新手最高频踩坑）**

    - 直接执行`sudo hostname 新主机名`仅修改**内核动态主机名**，重启后会被`/etc/hostname`中的静态主机名覆盖，绝对不要用这种方式做永久配置；

    - 永久修改必须用`hostnamectl set-hostname`或修改`/etc/hostname`文件，同时必须同步`/etc/hosts`。

2. **/etc/hosts不同步的致命坑**

    - 仅修改主机名，不更新`/etc/hosts`中IP与主机名的绑定关系，会导致：sudo命令执行卡顿、ssh登录缓慢、服务启动报错「无法解析主机名」、集群节点间通信失败；

    - 强制要求：任何主机名修改操作，必须同步更新`/etc/hosts`文件。

3. **主机名命名规范的坑**

    - 主机名必须符合RFC 1123规范，否则会导致网络解析异常、服务兼容性问题：

        - 仅允许使用：小写英文字母`a-z`、数字`0-9`、连字符`-`、点`.`；

        - 禁止使用：下划线`_`、空格、特殊字符`!@#$%^&*()`、大写字母（部分服务不兼容大写）；

        - 不能以连字符`-`开头或结尾，单段长度不超过63个字符，总长度不超过253个字符。

4. **云服务器主机名被重置的坑**

    - 阿里云、腾讯云等公有云服务器，默认cloud-init服务会在重启时重置主机名，仅修改`/etc/hostname`无效，必须修改`cloud.cfg`中的`preserve_hostname: true`，才能永久生效。

5. **容器环境主机名修改的坑**

    - Docker/Podman容器中，用`hostname`修改的主机名仅临时生效，容器重启后会失效；

    - 容器永久设置主机名，必须在创建容器时通过`--hostname`参数指定：`docker run -d --hostname mysql-node01 mysql:8.0`。

6. **sudo权限配置的坑**

    - 若主机名解析异常，会导致sudo命令执行时出现`unable to resolve host 主机名: Name or service not known`的报错，即使sudo配置正确也会卡顿，解决方法是在`/etc/hosts`中添加`127.0.0.1 你的主机名`。

## 七、同类命令对比&拓展

|命令|核心作用|与hostname的区别&适用场景|
|---|---|---|
|`hostnamectl`|systemd系统的主机名管理命令|是hostname的进阶替代方案，可管理静态/瞬态/灵活三种主机名，支持永久修改，现代Linux系统优先推荐，hostname更适合临时查看、兼容旧系统|
|`uname -n`|输出系统的节点名（与短主机名一致）|仅能查看主机名，无修改、FQDN、IP查询能力，适合脚本中极简获取主机名，功能单一|
|`dnsdomainname`|输出系统的DNS域名|等价于`hostname -d`，仅能查看域名后缀，无其他功能，已逐步被hostname替代|
|`nmcli`|NetworkManager网络管理命令|可通过网络配置修改主机名，适合网络配置一体化管理场景，hostname专注于主机名本身，更轻量化|
## 八、课后练习任务

1. 查看你当前系统的短主机名、FQDN、所有网卡IP地址，记录下来。

2. 用`hostnamectl`命令将系统主机名永久修改为`linux-node01`，同步更新`/etc/hosts`文件，验证重启后仍能生效。

3. 写一个简易脚本，动态获取主机名，生成格式为`log_主机名_年月日.log`的日志文件，并写入一条带主机名的测试日志。

4. 用之前的校验脚本，验证以下主机名是否符合规范：`test_node01`、`prod-node01`、`-test-node`、`K8S-MASTER`，并说明不符合的原因。

5. 排查主机名解析问题：模拟修改主机名后不更新`/etc/hosts`，执行sudo命令查看是否出现报错，然后修复该问题。
> （注：文档部分内容可能由 AI 生成）