
Facts 采集是 Play 执行的默认前置步骤，全量采集会增加剧本执行时间，在大规模集群（数百台以上节点）、短平快的剧本执行场景中，性能影响尤为明显。本节完整覆盖大纲指定的关闭与优化方案，实现执行性能的大幅提升。

### 5.7.1 关闭 Facts 采集

#### 1. Play 级关闭

在 Play 定义中设置`gather_facts: no`，即可完全关闭当前 Play 的 Facts 采集，是最常用的关闭方式。

**示例**：

```yaml
---
- name: 关闭Facts采集的Play
  hosts: all
  gather_facts: no  # 完全关闭Facts采集
  tasks:
    - name: 安装Nginx
      yum:
        name: nginx
        state: present
```

**适用场景**：剧本中未引用任何 Facts 变量，无需采集系统信息，可完全关闭，大幅缩短执行时间。

#### 2. 全局关闭

在`ansible.cfg`中设置全局默认关闭 Facts 采集，所有 Play 默认不采集 Facts，仅需在需要的 Play 中单独开启`gather_facts: yes`。

**配置示例**：

```bash
[defaults]
gathering = explicit  # 显式模式，默认关闭Facts采集，仅Play中设置gather_facts: yes时才采集
```

### 5.7.2 Facts 采集性能优化方案

若剧本中需要使用部分 Facts 变量，无需全量采集，可通过以下方案实现精准采集，兼顾功能需求与执行性能。

#### 1. 按需采集子集（gather_subset）

通过`gather_subset`参数，指定仅采集需要的 Facts 子集，跳过无关的采集项，大幅缩短采集时间。

**语法规则**：在 Play 中通过`gather_subset`字段指定采集子集，支持`!`排除指定子集。

**核心可选子集**：

|子集名|采集内容|
|:--|:--|
|`all`|全量采集，默认值|
|`min`|最小子集，仅采集操作系统、主机名、架构等核心信息|
|`network`|网络相关 Facts，网卡、IP、路由等|
|`hardware`|硬件相关 Facts，CPU、内存、磁盘等|
|`virtual`|虚拟化相关 Facts|
|`ohai`/`facter`|第三方采集工具数据|

**标准示例**：

```yaml
---
- name: 仅采集网络与硬件Facts
  hosts: all
  gather_facts: yes
  gather_subset:
    - network
    - hardware
  tasks:
    - name: 输出网卡信息
      debug:
        var: ansible_default_ipv4
```

**排除示例**：采集除了虚拟化之外的所有 Facts

```yaml
---
- name: 排除虚拟化Facts采集
  hosts: all
  gather_facts: yes
  gather_subset:
    - all
    - "!virtual"
```

#### 2. Facts 缓存机制

开启 Facts 缓存，将采集到的 Facts 信息持久化存储，后续执行剧本时直接从缓存中读取，无需重复采集，仅当缓存过期时才重新采集，适用于频繁执行的剧本、大规模集群场景。

**缓存配置（ansible.cfg）**：

1. JSON 文件缓存（最简单，无需额外依赖）：

    ```bash
    [defaults]
    # 开启Facts缓存
    gathering = smart
    # 缓存后端类型：jsonfile
    fact_caching = jsonfile
    # 缓存文件存储目录
    fact_caching_connection = /var/lib/ansible/facts_cache
    # 缓存过期时间，单位秒，86400=1天
    fact_caching_timeout = 86400
    ```
    
2. Redis 缓存（适用于分布式、高并发场景）：

    ```bash
    [defaults]
    gathering = smart
    fact_caching = redis
    fact_caching_connection = redis://127.0.0.1:6379/0
    fact_caching_timeout = 86400
    ```
    

**核心配置说明**：

- `gathering = smart`：智能模式，仅当缓存不存在或过期时，才重新采集 Facts，缓存有效时直接读取，是缓存模式的核心配置；
- 缓存过期时间可根据基础设施变化频率调整，静态环境可设置更长的过期时间。

#### 3. 手动按需采集 setup 模块

全局关闭 Facts 采集，仅在需要使用 Facts 的 Task 前，手动调用`setup`模块采集指定子集，实现极致的按需采集。

**示例**：

```yaml
---
- name: 全局关闭Facts，手动按需采集
  hosts: all
  gather_facts: no
  tasks:
    - name: 仅采集操作系统相关Facts
      setup:
        gather_subset:
          - min
          - "!all"

    - name: 基于操作系统执行对应任务
      yum:
        name: nginx
        state: present
      when: ansible_os_family == "RedHat"
```

### 5.7.3 优化最佳实践

1. **最小采集原则**：剧本中未使用任何 Facts 变量时，必须设置`gather_facts: no`，完全关闭采集；
2. **子集优先**：仅需部分 Facts 时，优先使用`gather_subset`精准采集，禁止全量采集；
3. **缓存适配**：频繁执行的剧本、大规模集群场景，必须开启 Facts 缓存，减少重复采集开销；
4. **避免过度依赖**：能通过 Inventory 变量、魔法变量实现的逻辑，优先不使用 Facts 变量，减少采集依赖。

---

## 本章小结

本章完整覆盖了 Ansible 动态化能力的全体系内容，严格遵循大纲顺序，从变量作用域与优先级规则、全场景变量定义方式，到 Register 结果捕获机制、Facts 系统信息采集、内置魔法变量，再到 Jinja2 模板的全语法体系与实战，最终完成了 Facts 采集的关闭与性能优化方案，无核心知识点遗漏。

本章内容是 Ansible 从静态剧本到动态、可复用、工程化自动化方案的核心转折点，解决了硬编码、环境适配、跨节点协同、动态配置生成等核心问题，是后续流程控制、角色化编排、企业级实战场景的核心前置基础。