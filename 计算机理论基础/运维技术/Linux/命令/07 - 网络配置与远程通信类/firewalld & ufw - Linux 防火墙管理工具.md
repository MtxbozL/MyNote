## 一、核心定位

`firewalld` 和 `ufw` 是 Linux 上**最常用的两大防火墙管理工具**，均为 `iptables`/`nftables` 的前端，简化了复杂的底层规则配置：

- **firewalld**：RedHat/CentOS/Fedora 等发行版的**默认动态防火墙**，支持 “区域（zones）” 概念，区分运行时 / 永久配置，功能更丰富。
- **ufw**（Uncomplicated Firewall）：Debian/Ubuntu 等发行版的**默认简单防火墙**，语法极简，适合快速配置日常规则。

---

## 二、firewalld 详解

### 1. 核心概念

- **区域（Zones）**：预定义的网络信任级别（如 `public`、`home`、`work`、`trusted`），不同区域有不同的默认规则，默认区域为 `public`。
- **运行时 / 永久配置**：
    
    - 不加 `--permanent`：仅修改**运行时**（立即生效，重启失效）。
    - 加 `--permanent`：仅修改**永久配置**（不立即生效，需 `--reload` 后生效，重启保留）。
    

### 2. 语法格式

```bash
firewall-cmd [选项]
```

### 3. 高频核心命令

| 功能           | 命令                                                                                       | 说明                           |
| :----------- | :--------------------------------------------------------------------------------------- | :--------------------------- |
| **状态与重载**    | `firewall-cmd --state`                                                                   | 查看防火墙状态（running/not running） |
|              | `firewall-cmd --reload`                                                                  | 重载永久配置，不中断连接                 |
|              | `firewall-cmd --list-all`                                                                | 查看当前区域的所有规则                  |
| **区域管理**     | `firewall-cmd --get-default-zone`                                                        | 查看默认区域                       |
|              | `firewall-cmd --set-default-zone=home`                                                   | 设置默认区域为 `home`               |
|              | `firewall-cmd --list-zones`                                                              | 查看所有可用区域                     |
| **服务管理**     | `firewall-cmd --add-service=ssh`                                                         | 运行时开放 `ssh` 服务（默认端口 22）      |
|              | `firewall-cmd --add-service=ssh --permanent`                                             | 永久开放 `ssh` 服务                |
|              | `firewall-cmd --remove-service=ssh`                                                      | 运行时移除 `ssh` 服务               |
|              | `firewall-cmd --list-services`                                                           | 查看当前区域开放的服务                  |
| **端口管理**     | `firewall-cmd --add-port=80/tcp`                                                         | 运行时开放 80/TCP 端口              |
|              | `firewall-cmd --add-port=80/tcp --permanent`                                             | 永久开放 80/TCP 端口               |
|              | `firewall-cmd --remove-port=80/tcp`                                                      | 运行时移除 80/TCP 端口              |
|              | `firewall-cmd --list-ports`                                                              | 查看当前区域开放的端口                  |
| **源地址 / 接口** | `firewall-cmd --add-source=192.168.1.0/24`                                               | 运行时允许 192.168.1.0/24 网段访问    |
|              | `firewall-cmd --add-interface=eth0`                                                      | 将 `eth0` 接口加入当前区域            |
| **富规则（高级）**  | `firewall-cmd --add-rich-rule='rule family="ipv4" source address="192.168.1.10" accept'` | 允许 192.168.1.10 的所有访问        |

---

## 三、ufw 详解

### 1. 核心概念

- **简单易用**：语法极简，无需复杂的区域或规则概念，适合快速配置。
- **默认状态**：多数发行版默认**关闭**，需先 `enable` 开启。
- **规则顺序**：按添加顺序匹配，先匹配先生效。

### 2. 语法格式

```bash
ufw [选项] 命令
```

### 3. 高频核心命令

| 功能            | 命令                                    | 说明                     |
| :------------ | :------------------------------------ | :--------------------- |
| **状态与开关**     | `ufw status`                          | 查看防火墙状态和规则             |
|               | `ufw enable`                          | 开启防火墙                  |
|               | `ufw disable`                         | 关闭防火墙                  |
|               | `ufw reload`                          | 重载规则                   |
| **默认策略**      | `ufw default deny incoming`           | 默认拒绝所有入站连接（推荐）         |
|               | `ufw default allow outgoing`          | 默认允许所有出站连接（推荐）         |
| **服务 / 端口管理** | `ufw allow ssh`                       | 允许 `ssh` 服务（默认端口 22）   |
|               | `ufw allow 80/tcp`                    | 允许 80/TCP 端口           |
|               | `ufw allow 53/udp`                    | 允许 53/UDP 端口（DNS）      |
|               | `ufw deny 80/tcp`                     | 拒绝 80/TCP 端口           |
|               | `ufw delete allow 80/tcp`             | 删除允许 80/TCP 的规则        |
| **源地址 / 接口**  | `ufw allow from 192.168.1.0/24`       | 允许 192.168.1.0/24 网段访问 |
|               | `ufw allow in on eth0 to any port 80` | 允许 `eth0` 接口的 80 端口入站  |
| **规则编号管理**    | `ufw status numbered`                 | 查看带编号的规则               |
|               | `ufw delete 2`                        | 删除编号为 2 的规则            |

---

## 四、避坑提示与注意事项

### firewalld

1. **运行时和永久分开，改完要 `--reload`**
    
    - 仅加 `--permanent` 不会立即生效，必须 `--reload`；
    - 推荐：先测试运行时，确认无误后再加 `--permanent` 并 `--reload`。
    
2. **区域概念很重要，不要乱改默认区**
    
    - 不同区域有不同的信任级别，`public` 是最常用的公共网络区域；
    - 不要随意将默认区改为 `trusted`（允许所有），风险极高。
    
3. **富规则复杂但强大，适合高级场景**
    
    - 富规则可以实现更精细的控制（如按源地址、协议、端口组合），但语法较复杂，需仔细测试。
    

### ufw

1. **默认可能关闭，要先 `enable`**
    
    - 多数发行版默认 `ufw` 是关闭的，配置规则后要 `ufw enable` 开启；
    - 开启前建议先配置好 `ssh` 规则，避免远程连接断开。
    
2. **规则简单但足够日常用**
    
    - `ufw` 语法极简，适合快速配置日常的服务 / 端口规则；
    - 高级场景可配合 `iptables` 或切换到 `firewalld`。
    
3. **删除规则用编号更方便**
    
    - `ufw status numbered` 查看带编号的规则，`ufw delete 编号` 直接删除，避免记错规则内容。
    

---

## 五、同类对比

表格

|特性|firewalld|ufw|
|:--|:--|:--|
|**默认发行版**|RHEL/CentOS/Fedora|Debian/Ubuntu|
|**核心概念**|动态、区域（zones）|简单、静态规则|
|**配置复杂度**|中等（区域 + 富规则）|低（极简语法）|
|**运行时 / 永久**|区分，需 `--reload`|不区分，直接生效|
|**适用场景**|复杂网络环境、多区域|日常简单配置、快速上手|

---

## 六、课后练习

### firewalld 练习

1. 查看防火墙状态和当前区域的所有规则。
2. 设置默认区域为 `home`，再改回 `public`。
3. 运行时开放 `http`（80/TCP）和 `https`（443/TCP）服务，测试后永久开放并 `--reload`。
4. 运行时开放 3306/TCP 端口（MySQL），测试后移除。
5. 允许 192.168.1.0/24 网段访问所有服务。

### ufw 练习

1. 查看防火墙状态，开启防火墙。
2. 设置默认策略：拒绝所有入站，允许所有出站。
3. 允许 `ssh`（22/TCP）、`http`（80/TCP）、`https`（443/TCP）服务。
4. 允许 53/UDP 端口（DNS）。
5. 查看带编号的规则，删除其中一条规则。
6. 允许 192.168.1.0/24 网段访问。