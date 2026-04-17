
>本章完整覆盖 Ansible Playbook 的核心语法体系与工程化基础，建立声明式自动化剧本的标准化编写能力。内容从 YAML 语法的 Ansible 适配规范、Playbook 核心层级结构，到首个标准化剧本的编写与解析、全场景提权机制、全链路执行调试工具，最终完成幂等性的实战落地与最佳实践，无核心知识点遗漏。Playbook 是 Ansible 区别于单任务 Ad-Hoc 命令的核心载体，是企业级自动化流程编排、版本控制、可复用能力建设的核心基石。

---

## 4.1 YAML 语法速成（Ansible 适配版）

YAML（YAML Ain't Markup Language）是一种人类可读的、基于缩进的结构化数据序列化语言，是 Ansible Playbook 的 **唯一标准语法格式** ，核心设计目标是易读、易写、易维护，完全适配声明式配置管理场景。本节仅聚焦 Ansible 编写必需的 YAML 核心规则，无冗余内容。

### 4.1.1 核心语法规则

#### 1. 缩进与格式规范

- Ansible 强制使用 **2 个空格** 作为单层级缩进，**绝对禁止使用 Tab 制表符**，同一层级的元素必须保持严格左对齐，缩进是 YAML 唯一的层级标识；
- 语法大小写敏感，所有关键字、变量名、参数名、模块名均严格区分大小写；
- 键值对格式为`key: value`，冒号后必须跟随一个空格，否则会触发语法错误。

#### 2. 注释规则

- 以`#`开头标识注释，支持行首注释与行尾注释，注释内容不会被 Ansible 解析；
- 仅支持单行注释，多行注释每行均需以`#`开头，无块注释语法。

```yaml
# 这是行首注释
- name: 安装Nginx  # 这是行尾注释
  yum:
    name: nginx
    state: present
```

#### 3. 列表（Sequence）

对应 Python 的列表数据结构，用于表示有序元素集合，Ansible 中用于定义`tasks`、`hosts`、`tags`等序列数据，支持两种写法：

- **短横线分行写法（Ansible 官方推荐）**：每行以`-` （短横线 + 空格）开头，同一缩进层级，可读性强，是 Playbook 的标准写法；
- **方括号内联写法**：`[元素1, 元素2, 元素3]`，仅适用于短列表场景。

```yaml
# 标准分行写法
tasks:
  - name: 安装Nginx
    yum: name=nginx state=present
  - name: 启动Nginx
    service: name=nginx state=started enabled=yes

# 内联写法
hosts: [webservers, dbservers]
```

#### 4. 字典（Mapping）

对应 Python 的字典数据结构，为键值对格式，用于表示无序属性集合，Ansible 中用于定义 Play、Task、模块参数等配置，支持两种写法：

- **冒号分行写法（Ansible 官方推荐）**：同一缩进层级下分行定义键值对，可读性强，是 Playbook 的标准写法；
- **大括号内联写法**：`{key1: value1, key2: value2}`，仅适用于短属性集合场景。

```yaml
# 标准分行写法
- name: 配置防火墙
  firewalld:
    zone: public
    service: http
    permanent: yes
    state: enabled
    immediate: yes

# 内联写法
- name: 安装Nginx
  yum: {name: nginx, state: present}
```

#### 5. 字符串与标量处理

- 普通纯文本字符串无需引号包裹；
- 字符串包含特殊字符（冒号、&、*、#、| 等）、空格、换行符时，必须用单引号或双引号包裹；
    
    - 单引号：纯字面量解析，不转义任何字符，包括反斜杠`\`；
    - 双引号：支持转义字符解析，如`\n`（换行）、`\t`（制表符）、`\"`（双引号转义）；
    
- 多行字符串处理（Ansible 高频使用场景）：
    
    - `|` 保留换行符：多行文本的换行符会被完整保留，适用于脚本、配置文件内容定义；
    - `>` 折叠换行符：多行文本的换行符会被替换为单个空格，适用于长文本描述。
    

```yaml
# 保留换行符示例
- name: 写入自定义配置
  copy:
    content: |
      server {
          listen 80;
          server_name localhost;
          root /usr/share/nginx/html;
          index index.html;
      }
    dest: /etc/nginx/conf.d/default.conf

# 折叠换行符示例
- name: 任务描述
  debug:
    msg: >
      这是一段超长的任务描述文本，
      会被折叠为单行字符串，
      换行符会被替换为空格。
```

#### 6. 布尔值规范

Ansible 全版本兼容的布尔值写法为`yes/no`、`true/false`，**生产环境推荐统一使用小写的 yes/no**，避免跨版本、跨平台的兼容问题。

### 4.1.2 高频语法踩坑点

1. 冒号后未加空格，是最常见的语法错误；
2. 混用 Tab 与空格缩进，导致 Ansible 解析层级异常；
3. 字符串包含冒号未加引号包裹，被误解析为键值对；
4. 同一层级元素缩进不一致，导致结构解析错误。

---
