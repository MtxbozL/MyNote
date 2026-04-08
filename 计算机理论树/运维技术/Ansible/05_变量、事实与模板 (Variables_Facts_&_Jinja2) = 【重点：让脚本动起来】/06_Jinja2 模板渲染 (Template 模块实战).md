### 5.6.1 核心定义与执行原理

Jinja2 是 Python 生态的高性能模板引擎，是 Ansible 实现动态配置生成的核心载体，与 Ansible 的变量体系深度集成，支持变量替换、条件判断、循环、过滤器等全功能语法，可动态生成任意文本格式的配置文件、脚本、清单等。

**核心执行原理**：

1. Ansible 的`template`模块在**控制节点本地**完成 Jinja2 模板的渲染，将变量、逻辑替换为最终的文本内容；
2. 渲染完成后，将最终的文本文件复制到被控节点的目标路径；
3. 被控节点无需安装 Jinja2 依赖，仅控制节点需要（Ansible 安装时已默认集成）；
4. 模板渲染完全遵循幂等性，仅当渲染后的文件内容与被控节点目标文件不一致时，才会执行覆盖操作。

### 5.6.2 基础语法与变量替换

#### 1. 核心分隔符

Jinja2 通过三种核心分隔符区分模板语法与普通文本，Ansible 中使用默认分隔符，无需自定义：

|分隔符|核心作用|示例|
|:--|:--|:--|
|`{{ 变量/表达式 }}`|变量替换 / 表达式输出，将括号内的内容渲染为最终文本|`{{ nginx_port }}`|
|`{% 控制语句 %}`|逻辑控制语句，如 if 条件判断、for 循环、宏定义|`{% if nginx_ssl_enable %}`|
|`{# 注释内容 #}`|模板注释，渲染后不会出现在最终文件中|`{# 这是模板注释，不会输出 #}`|

#### 2. 基础变量替换

模板中直接通过`{{ 变量名 }}`引用 Ansible 的所有变量，包括 Play 变量、Inventory 变量、Facts、魔法变量、Register 变量等，支持层级字典的点号引用`{{ 字典.键.子键 }}`。

**标准模板示例**：`nginx.conf.j2`

```jinja2
{# Nginx基础配置模板，支持变量动态渲染 #}
user nginx;
worker_processes {{ nginx_config.worker_processes | default('auto') }};
error_log /var/log/nginx/error.log;
pid /var/run/nginx.pid;

events {
    worker_connections {{ nginx_config.worker_connections | default(1024) }};
    use epoll;
    multi_accept on;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    server {
        listen       {{ nginx_port | default(80) }};
        server_name  {{ inventory_hostname }} {{ ansible_default_ipv4.address }};
        root         {{ nginx_root }};
        index        index.html index.htm;
    }
}
```

**Playbook 调用示例**：

```yaml
---
- name: 渲染Nginx配置模板
  hosts: webservers
  become: yes
  vars:
    nginx_port: 80
    nginx_root: /usr/share/nginx/html
    nginx_config:
      worker_processes: auto
      worker_connections: 2048
  tasks:
    - name: 渲染模板并分发
      template:
        src: ./templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        mode: 0644
        owner: root
        group: root
      notify: 重启Nginx

  handlers:
    - name: 重启Nginx
      service:
        name: nginx
        state: restarted
```

### 5.6.3 控制结构

Jinja2 提供完整的流程控制语法，实现配置文件的动态生成，核心覆盖大纲指定的`for`循环与`if`条件判断。

#### 1. if 条件判断

**语法规则**：通过`{% if 条件 %}`、`{% elif 条件 %}`、`{% else %}`、`{% endif %}`实现分支逻辑，条件表达式支持 Ansible 变量、比较运算符、逻辑运算符。

**标准示例**：基于变量动态开启 SSL 配置

```jinja2
server {
    listen 80;
    server_name {{ server_name }};

    {% if nginx_ssl_enable | default(false) %}
    # 自动跳转HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name {{ server_name }};

    ssl_certificate {{ ssl_cert_path }};
    ssl_certificate_key {{ ssl_key_path }};
    ssl_protocols TLSv1.2 TLSv1.3;
    {% else %}
    root {{ nginx_root }};
    index index.html;
    {% endif %}
}
```

**进阶示例**：基于操作系统发行版动态配置

```jinja2
{% if ansible_os_family == "RedHat" %}
pid /var/run/nginx.pid;
{% elif ansible_os_family == "Debian" %}
pid /run/nginx.pid;
{% else %}
pid /var/run/nginx.pid;
{% endif %}
```

#### 2. for 循环

**语法规则**：通过`{% for 变量 in 可迭代对象 %}`、`{% endfor %}`实现循环遍历，支持列表、字典、Ansible 分组列表等可迭代对象，可配合`if`过滤、`loop.index`循环索引、`loop.first`/`loop.last`首尾判断。

**标准示例 1**：遍历列表生成 location 配置

```jinja2
{% for path in static_paths %}
location {{ path }} {
    root {{ nginx_root }};
    expires 7d;
    add_header Cache-Control "public";
}
{% endfor %}
```

对应 Play 变量：

```yaml
static_paths:
  - /static
  - /images
  - /css
  - /js
```

**标准示例 2**：遍历主机组生成 upstream 集群配置

```jinja2
upstream backend_cluster {
    least_conn;
    {% for host in groups['webservers'] %}
    server {{ hostvars[host].ansible_default_ipv4.address }}:80 max_fails=3 fail_timeout=30s;
    {% endfor %}
}
```

**标准示例 3**：遍历字典生成配置项

```jinja2
{% for key, value in nginx_config.items() %}
{{ key }} {{ value }};
{% endfor %}
```

对应 Play 变量：

```yaml
nginx_config:
  sendfile: "on"
  tcp_nopush: "on"
  tcp_nodelay: "on"
  keepalive_timeout: 65
  gzip: "on"
```

**循环内置变量**：

|变量|核心含义|
|:--|:--|
|`loop.index`|当前循环次数，从 1 开始|
|`loop.index0`|当前循环次数，从 0 开始|
|`loop.first`|布尔值，是否为第一次循环|
|`loop.last`|布尔值，是否为最后一次循环|
|`loop.length`|循环总次数|

### 5.6.4 过滤器

过滤器是 Jinja2 提供的变量处理工具，通过`|`管道符调用，用于对变量的值进行格式化、转换、过滤、计算等处理，是模板渲染中数据处理的核心能力。本节严格覆盖大纲指定的`default`、`json_query`、`regex_replace`过滤器，同时补充生产环境高频常用过滤器。

#### 1. 核心基础过滤器

|过滤器|核心作用|标准示例||
|:--|:--|:--|---|
|`default`|设置变量默认值，当变量未定义或为空时生效，避免变量未定义报错|`{{ nginx_port|default(80) }}`|
|`mandatory`|强制变量必须定义，未定义则直接报错，用于必填参数校验|`{{ db_password|mandatory (' 数据库密码必须配置 ') }}`|
|`lower/upper`|字符串转换为小写 / 大写|`{{ ansible_hostname|upper }}`|
|`trim`|去除字符串首尾的空白字符|`{{ command_output|trim }}`|
|`replace`|字符串替换|`{{ path|replace('/old', '/new') }}`|
|`regex_replace`|正则表达式替换，大纲指定核心过滤器|`{{ version|regex_replace('^v', '') }}`|
|`int/float`|转换为整数 / 浮点数|`{{ worker_count|int }}`|
|`string`|转换为字符串|`{{ port|string }}`|
|`bool`|转换为布尔值|`{{ enable|bool }}`|
|`to_json/to_yaml`|变量转换为 JSON/YAML 格式字符串|`{{ config|to_json }}`|
|`from_json/from_yaml`|JSON/YAML 字符串转换为 Ansible 变量|`{{ json_output|from_json }}`|
|`sort`|列表排序|`{{ host_list|sort }}`|
|`unique`|列表去重|`{{ ip_list|unique }}`|
|`join`|列表拼接为字符串|`{{ dns_servers|join(', ') }}`|
|`split`|字符串拆分为列表|`{{ path|split('/') }}`|
|`json_query`|基于 JMESPath 语法查询复杂 JSON 数据，大纲指定核心过滤器|`{{ result|json_query('items[].status') }}`|

#### 2. 大纲指定核心过滤器详解

##### （1）default 过滤器

**核心作用**：为变量设置兜底默认值，解决变量未定义导致的剧本执行失败问题，是最常用的过滤器。

**语法规则**：`{{ 变量名 | default(默认值, boolean=是否校验布尔值) }}`

- 第二个参数`boolean=false`（默认）：仅当变量未定义时，使用默认值；变量为空字符串、0、false 时，仍使用变量原值；
- 第二个参数`boolean=true`：变量未定义、为空、0、false 时，均使用默认值。

**示例**：

```jinja2
# 变量未定义时，使用默认值80
listen {{ nginx_port | default(80) }};

# 变量为false/空时，也使用默认值
enable_ssl {{ nginx_ssl | default(false, boolean=true) }};
```

##### （2）regex_replace 过滤器

**核心作用**：基于正则表达式实现复杂的字符串替换、提取、格式化，适用于处理命令输出、版本号、路径等复杂字符串。

**语法规则**：`{{ 字符串 | regex_replace('正则表达式', '替换内容') }}`，支持正则分组引用`\1`、`\2`等。

**示例**：

```jinja2
# 去除版本号前缀的v，如v1.24.0 → 1.24.0
nginx_version: {{ nginx_version | regex_replace('^v', '') }}

# 提取IP地址中的网段，如192.168.1.101 → 192.168.1.0
network_segment: {{ ansible_default_ipv4.address | regex_replace('(\d+\.\d+\.\d+)\.\d+', '\1.0') }}

# 替换路径中的反斜杠为正斜杠
unix_path: {{ windows_path | regex_replace('\\\\', '/') }}
```

##### （3）json_query 过滤器

**核心作用**：基于 JMESPath 查询语法，从复杂的嵌套 JSON / 字典数据中精准提取目标字段，适用于处理 API 返回结果、Register 变量的复杂结构化数据，无需多层循环嵌套。

**前置依赖**：使用前需在控制节点安装`jmespath`库：`pip install jmespath`。

**语法规则**：`{{ 复杂变量 | json_query('JMESPath查询表达式') }}`

**示例**：

1. 基础嵌套查询：

    ```yaml
    # 原始变量
    app_result:
      code: 200
      data:
        app_name: nginx
        version: 1.24.0
        status: running
        ports:
          - 80
          - 443
    ```
    
    模板查询：

    ```jinja2
    # 提取应用版本号
    app_version: {{ app_result | json_query('data.version') }}
    
    # 提取端口列表
    app_ports: {{ app_result | json_query('data.ports') }}
    ```
    
2. 列表过滤查询：

    ```yaml
    # 原始变量
    nodes:
      - name: web01
        ip: 192.168.1.101
        role: web
        status: ready
      - name: db01
        ip: 192.168.1.201
        role: db
        status: ready
      - name: web02
        ip: 192.168.1.102
        role: web
        status: down
    ```
    
    模板查询：

    ```jinja2
    # 提取所有web角色节点的IP地址
    web_ips: {{ nodes | json_query("[?role=='web'].ip") }}
    
    # 提取所有ready状态的节点名称
    ready_nodes: {{ nodes | json_query("[?status=='ready'].name") }}
    ```
    

---
