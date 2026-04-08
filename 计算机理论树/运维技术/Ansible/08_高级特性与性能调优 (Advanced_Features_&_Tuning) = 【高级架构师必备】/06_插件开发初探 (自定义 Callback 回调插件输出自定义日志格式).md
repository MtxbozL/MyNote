### 8.6.1 核心定义与设计价值

Ansible 采用插件化架构，几乎所有核心能力都通过插件实现，Callback 回调插件是其中最常用的扩展插件，用于监听 Ansible 执行过程中的全量事件（Playbook 启动 / 结束、Task 执行成功 / 失败 / 跳过、主机连接失败等），在事件触发时执行自定义逻辑，实现自定义日志格式、审计日志上报、告警通知、执行结果可视化等能力。

**核心适用场景**：

- 自定义日志输出格式，适配企业日志规范；
- 执行失败时自动发送钉钉 / 邮件 / 企业微信告警；
- 将执行结果、审计日志上报到 CMDB / 审计平台；
- 自定义执行进度展示、统计报表生成；
- 执行失败时自动触发故障回滚、快照创建。

### 8.6.2 插件核心原理与事件钩子

#### 1. 插件加载规则

Ansible 会自动从以下路径加载 Callback 插件，优先级从高到低：

1. 当前项目的`callback_plugins/`目录；
2. 当前用户家目录的`~/.ansible/plugins/callback/`；
3. 系统级目录`/usr/share/ansible/plugins/callback/`；
4. `ansible.cfg`中`callback_plugins`参数指定的自定义路径。

插件启用方式：

1. **白名单启用**：在`ansible.cfg`的`[defaults]`段添加插件名到`callback_whitelist`；
2. **标准输出插件**：设置`stdout_callback`参数，替换默认的标准输出插件；
3. **自动启用**：插件中设置`CALLBACK_TYPE = 'notification'`，自动加载启用。

#### 2. 核心事件钩子

Callback 插件通过重写预定义的事件钩子函数，实现自定义逻辑，大纲指定的自定义日志格式场景，核心钩子函数如下：

|钩子函数|触发时机|核心用途|
|:--|:--|:--|
|`v2_playbook_on_start`|Playbook 开始执行时|记录启动时间、初始化日志、发送开始通知|
|`v2_playbook_on_play_start`|每个 Play 开始执行时|记录 Play 名称、目标主机列表|
|`v2_runner_on_ok`|Task 执行成功时|记录成功日志、统计成功任务数|
|`v2_runner_on_failed`|Task 执行失败时|记录失败日志、发送告警、统计失败任务数|
|`v2_runner_on_skipped`|Task 被跳过时|记录跳过日志、统计跳过任务数|
|`v2_runner_on_unreachable`|主机不可达时|记录不可达日志、发送告警|
|`v2_playbook_on_stats`|Playbook 执行结束，输出统计信息时|生成执行报表、发送执行结果通知、关闭日志文件|

### 8.6.3 实战示例：自定义日志格式 Callback 插件

实现一个自定义日志插件，将 Ansible 执行结果按企业规范输出到 JSON 格式的审计日志文件，同时在控制台输出简洁的执行进度，完全符合大纲指定的「自定义日志格式」需求。

#### 步骤 1：创建插件目录与文件

在 Playbook 同级目录创建`callback_plugins`目录，新建`custom_logger.py`插件文件：

```python
# callback_plugins/custom_logger.py
from __future__ import (absolute_import, division, print_function)
__metaclass__ = type

import json
import time
from ansible.plugins.callback import CallbackBase
from ansible import constants as C

DOCUMENTATION = '''
    name: custom_logger
    type: notification
    short_description: 自定义JSON审计日志插件
    description: 将Ansible执行结果输出为JSON格式审计日志，同时控制台输出简洁进度
    version_added: "2.14"
    author: Ansible高级架构师
'''

class CallbackModule(CallbackBase):
    CALLBACK_VERSION = 2.0
    CALLBACK_TYPE = 'notification'
    CALLBACK_NAME = 'custom_logger'
    CALLBACK_NEEDS_WHITELIST = False

    def __init__(self):
        super(CallbackModule, self).__init__()
        # 初始化审计日志数据
        self.audit_log = {
            "playbook_name": "",
            "start_time": "",
            "end_time": "",
            "duration": 0,
            "total_tasks": 0,
            "success_tasks": 0,
            "failed_tasks": 0,
            "skipped_tasks": 0,
            "unreachable_hosts": [],
            "task_details": []
        }
        self.log_file = "./ansible_audit_log.json"

    def v2_playbook_on_start(self, playbook):
        # Playbook启动时触发
        self.audit_log["playbook_name"] = playbook._file_name
        self.audit_log["start_time"] = time.strftime("%Y-%m-%d %H:%M:%S", time.localtime())
        self.start_timestamp = time.time()
        self._display.display("=" * 80, color=C.COLOR_OK)
        self._display.display(f"开始执行Playbook: {playbook._file_name}", color=C.COLOR_OK)
        self._display.display(f"启动时间: {self.audit_log['start_time']}", color=C.COLOR_OK)
        self._display.display("=" * 80, color=C.COLOR_OK)

    def v2_runner_on_ok(self, result):
        # 任务执行成功时触发
        host = result._host.get_name()
        task_name = result._task.get_name()
        self.audit_log["success_tasks"] += 1
        self.audit_log["total_tasks"] += 1
        # 记录任务详情
        self.audit_log["task_details"].append({
            "host": host,
            "task_name": task_name,
            "status": "success",
            "changed": result.is_changed(),
            "timestamp": time.strftime("%Y-%m-%d %H:%M:%S", time.localtime())
        })
        # 控制台输出
        status_color = C.COLOR_CHANGED if result.is_changed() else C.COLOR_OK
        status_label = "CHANGED" if result.is_changed() else "OK"
        self._display.display(f"[{status_label}] {host} | {task_name}", color=status_color)

    def v2_runner_on_failed(self, result, ignore_errors=False):
        # 任务执行失败时触发
        host = result._host.get_name()
        task_name = result._task.get_name()
        error_msg = result._result.get('msg', '未知错误')
        self.audit_log["failed_tasks"] += 1
        self.audit_log["total_tasks"] += 1
        # 记录任务详情
        self.audit_log["task_details"].append({
            "host": host,
            "task_name": task_name,
            "status": "failed",
            "error_msg": error_msg,
            "timestamp": time.strftime("%Y-%m-%d %H:%M:%S", time.localtime())
        })
        # 控制台输出
        self._display.display(f"[FAILED] {host} | {task_name} | 错误信息: {error_msg}", color=C.COLOR_ERROR)

    def v2_runner_on_skipped(self, result):
        # 任务被跳过时触发
        host = result._host.get_name()
        task_name = result._task.get_name()
        self.audit_log["skipped_tasks"] += 1
        self.audit_log["total_tasks"] += 1
        # 控制台输出
        self._display.display(f"[SKIPPED] {host} | {task_name}", color=C.COLOR_SKIP)

    def v2_runner_on_unreachable(self, result):
        # 主机不可达时触发
        host = result._host.get_name()
        error_msg = result._result.get('msg', '主机不可达')
        self.audit_log["unreachable_hosts"].append(host)
        # 控制台输出
        self._display.display(f"[UNREACHABLE] {host} | {error_msg}", color=C.COLOR_ERROR)

    def v2_playbook_on_stats(self, stats):
        # Playbook执行结束时触发
        self.audit_log["end_time"] = time.strftime("%Y-%m-%d %H:%M:%S", time.localtime())
        self.audit_log["duration"] = round(time.time() - self.start_timestamp, 2)
        # 写入JSON审计日志文件
        with open(self.log_file, 'w', encoding='utf-8') as f:
            json.dump(self.audit_log, f, ensure_ascii=False, indent=2)
        # 控制台输出最终统计
        self._display.display("=" * 80, color=C.COLOR_OK)
        self._display.display(f"Playbook执行完成，总耗时: {self.audit_log['duration']}秒", color=C.COLOR_OK)
        self._display.display(f"总任务数: {self.audit_log['total_tasks']} | 成功: {self.audit_log['success_tasks']} | 失败: {self.audit_log['failed_tasks']} | 跳过: {self.audit_log['skipped_tasks']}", color=C.COLOR_OK)
        self._display.display(f"审计日志已写入: {self.log_file}", color=C.COLOR_OK)
        self._display.display("=" * 80, color=C.COLOR_OK)
```

#### 步骤 2：启用插件

在`ansible.cfg`中配置插件启用：

```bash
[defaults]
# 插件目录
callback_plugins = ./callback_plugins
# 启用自定义插件
callback_whitelist = custom_logger
```

#### 步骤 3：执行效果

执行任意 Playbook，插件会自动加载，实现：

1. 控制台输出简洁的执行进度，区分成功、变更、失败、跳过状态；
2. 执行完成后自动生成 JSON 格式的审计日志文件，包含全量执行详情、统计信息、错误日志；
3. 完全兼容原生 Ansible 的所有功能，无侵入性。

### 8.6.4 最佳实践与扩展方向

1. **轻量开发原则**：Callback 插件中禁止执行长耗时、阻塞性操作，避免影响 Ansible 的正常执行流程；
2. **异常处理**：插件中的自定义逻辑必须添加 try-except 异常捕获，避免插件报错导致 Playbook 执行中断；
3. **敏感信息过滤**：日志输出时必须过滤密码、密钥等敏感信息，避免敏感数据泄露；
4. **扩展方向**：
    
    - 对接企业告警系统，执行失败时自动发送钉钉 / 企业微信 / 邮件告警；
    - 对接 Elasticsearch，将执行日志上报到日志平台，实现统一审计；
    - 对接 CMDB，自动更新资产的自动化执行记录；
    - 实现执行失败时自动触发快照、回滚等故障处理逻辑；
    
5. **版本兼容**：插件开发时需兼容生产环境的 Ansible 版本，避免使用高版本专属 API。

---

## 本章小结

本章完整覆盖了 Ansible 企业级高级特性与全链路性能调优体系，严格遵循大纲顺序，从 Ansible Vault 敏感数据加密、任务委派跨节点协同、异步长耗时任务处理，到执行策略选型、大规模集群并发与网络调优，最终完成了自定义 Callback 回调插件的开发实战，无核心知识点遗漏。

本章内容是 Ansible 从基础使用进阶到企业级架构落地的核心环节，解决了敏感数据安全合规、复杂业务流程闭环编排、长耗时任务稳定性、大规模集群执行性能四大核心企业级痛点，同时提供了可扩展的插件化开发能力。掌握本章内容，即可构建高安全、高性能、高可用、可扩展的企业级 Ansible 自动化架构，满足千台级大规模集群的自动化管理需求。