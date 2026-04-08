
本节基于生产环境最佳实践，编写一个完整可运行的 Nginx 服务部署 Playbook，覆盖核心语法规则，同时衔接第三章的模块知识，逐行解析核心逻辑。

### 4.3.1 完整 Playbook 示例

文件名：`nginx_deploy.yml`

```yaml
---
- name: 标准化部署Nginx Web服务
  hosts: webservers
  remote_user: ops
  become: yes
  gather_facts: yes

  tasks:
    - name: 安装Nginx软件包
      yum:
        name: nginx
        state: present

    - name: 配置Nginx默认站点
      copy:
        content: |
          server {
              listen 80 default_server;
              server_name _;
              root /usr/share/nginx/html;
              index index.html;
          }
        dest: /etc/nginx/conf.d/default.conf
        mode: 0644
        owner: root
        group: root
        backup: yes

    - name: 创建网站根目录
      file:
        path: /usr/share/nginx/html
        state: directory
        mode: 0755
        owner: nginx
        group: nginx
        recurse: yes

    - name: 写入测试首页
      copy:
        content: "<h1>Ansible自动化部署Nginx成功</h1><p>主机名: {{ ansible_hostname }}</p>"
        dest: /usr/share/nginx/html/index.html
        mode: 0644

    - name: 放行防火墙HTTP端口
      firewalld:
        zone: public
        service: http
        permanent: yes
        state: enabled
        immediate: yes

    - name: 启动Nginx服务并设置开机自启
      service:
        name: nginx
        state: started
        enabled: yes
```

### 4.3.2 逐行解析

1. **文档头与 Play 定义**：`---`标识 YAML 文档开始，`name`字段定义 Play 的业务含义，`hosts: webservers`指定目标主机为 Inventory 中的 webservers 分组；
2. **全局连接与提权配置**：`remote_user: ops`指定远程连接用户为 ops，`become: yes`开启全局 sudo 提权，所有任务默认以 root 权限执行；
3. **Facts 采集**：`gather_facts: yes`开启系统信息采集，后续任务通过`{{ ansible_hostname }}`变量调用主机名信息；
4. **任务列表**：按业务执行顺序定义 6 个 Task，每个 Task 调用一个官方模块，声明目标状态，完全遵循幂等性原则；
5. **变量渲染**：通过`{{ ansible_hostname }}`Jinja2 变量语法，动态插入被控节点的主机名，实现配置的动态生成。

### 4.3.3 Playbook 执行命令

```bash
# 基础执行命令
ansible-playbook nginx_deploy.yml

# 指定Inventory文件执行
ansible-playbook -i ./inventory/hosts nginx_deploy.yml
```

执行流程：

1. Ansible 加载配置文件与 Inventory 资产清单，匹配 webservers 分组的主机列表；
2. 解析 Playbook 语法，校验合法性；
3. 与目标主机建立 SSH 连接，采集 Facts 系统信息；
4. 按从上到下的顺序串行执行每个 Task，每个 Task 在所有目标主机执行完成后，才会进入下一个 Task；
5. 执行完成后，输出全量执行结果统计，包括成功、失败、变更、跳过的任务数量。

---
