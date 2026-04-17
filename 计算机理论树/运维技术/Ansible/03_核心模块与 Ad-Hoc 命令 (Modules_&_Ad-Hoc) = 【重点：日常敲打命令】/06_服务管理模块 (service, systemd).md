
>服务管理模块是 Ansible 管理被控节点系统服务的核心工具，覆盖服务的启动、停止、重启、重载、开机自启管理，分为通用 service 模块与 systemd 专属模块，所有模块均 **内置完整幂等性** 。

### 3.6.1 service 模块

**核心定义**：Ansible 通用服务管理模块，兼容 systemd、SysVinit、Upstart、OpenRC 等所有主流服务管理系统，自动适配被控节点的系统，是跨平台服务管理的通用方案。

#### 核心参数

| 参数        | 核心作用                                         | 必填项 |
| :-------- | :------------------------------------------- | :-- |
| `name`    | 服务名称，如 nginx、sshd                            | 是   |
| `state`   | 服务运行状态，可选 started/stopped/restarted/reloaded | 否   |
| `enabled` | 是否设置开机自启，布尔值 yes/no                          | 否   |
| `use`     | 指定服务管理系统，默认 auto 自动适配                        | 否   |

#### 标准示例

```bash
# 1. 启动nginx服务，并设置开机自启
ansible webservers -m service -a "name=nginx state=started enabled=yes" -b

# 2. 停止服务，关闭开机自启
ansible webservers -m service -a "name=nginx state=stopped enabled=no" -b

# 3. 重载服务配置
ansible webservers -m service -a "name=nginx state=reloaded" -b

# 4. 重启服务
ansible webservers -m service -a "name=nginx state=restarted" -b
```

#### 核心特性与注意事项

1. 必须提权执行；
2. 幂等性说明：`state=started/stopped` 是完全幂等的，仅当服务状态与目标不一致时才执行操作；`state=restarted`每次执行都会重启服务，非幂等，生产环境推荐使用 handlers 触发器，仅当配置变更时才重启；
3. 跨平台兼容性强，无需关注被控节点的服务管理系统，是通用服务管理的首选模块。

### 3.6.2 systemd 模块

**核心定义**：专门针对 systemd 系统的服务管理模块，提供比 service 模块更丰富的 systemd 专属功能，适配所有使用 systemd 的现代 Linux 发行版。

#### 核心参数

核心参数与 service 模块一致，额外支持 systemd 专属参数：

- `daemon_reload`：是否执行 systemctl daemon-reload，重载 systemd 配置，新增 / 修改.service 文件后必须执行，默认 no；
- `masked`：是否屏蔽服务，yes = 彻底屏蔽服务，无法启动，默认 no；
- `scope`：服务作用域，可选 system、user、global，默认 system。

#### 标准示例

```bash
# 1. 启动服务，设置开机自启，重载systemd配置
ansible webservers -m systemd -a "name=nginx.service state=started enabled=yes daemon_reload=yes" -b

# 2. 屏蔽firewalld服务，禁止启动
ansible all -m systemd -a "name=firewalld.service masked=yes state=stopped" -b

# 3. 仅重载systemd配置
ansible all -m systemd -a "daemon_reload=yes" -b
```

#### 核心注意事项

1. 仅适用于 systemd 系统，非 systemd 系统会执行失败，跨平台场景优先使用 service 模块；
2. 新增 / 修改.service 文件后，必须设置`daemon_reload=yes`，否则 systemd 无法识别新配置，是高频踩坑点；
3. `masked=yes`会彻底屏蔽服务，即使手动执行 systemctl start 也无法启动，用于禁用不需要的服务，提升系统安全性。

---
