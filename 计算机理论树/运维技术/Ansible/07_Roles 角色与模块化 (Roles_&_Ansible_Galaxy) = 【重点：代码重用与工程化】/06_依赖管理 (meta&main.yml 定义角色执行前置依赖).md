
角色依赖管理是指在角色的`meta/main.yml`文件中，通过`dependencies`字段定义该角色执行前必须提前执行的其他角色，Ansible 执行该角色时，会自动按顺序执行其定义的所有依赖角色，无需在 Playbook 中手动调用，实现角色间的依赖解耦与自动化执行。

### 7.6.1 核心语法与标准示例

以`php-fpm`角色为例，其依赖`nginx`角色，必须先安装 Nginx，再安装 PHP-FPM，依赖定义如下：

**文件路径**：`roles/php-fpm/meta/main.yml`

```yaml
---
galaxy_info:
  author: 示例作者
  description: PHP-FPM部署角色
  license: MIT
  min_ansible_version: 2.14
  platforms:
    - name: EL
      versions:
        - 8
        - 9
# 核心：角色依赖定义
dependencies:
  # 依赖nginx角色，指定版本与变量
  - role: nginx
    version: 3.2.0
    # 传递给依赖角色的变量
    vars:
      nginx_enable_php: yes
      nginx_listen_port: 80
  # 可定义多个依赖，按从上到下的顺序执行
  - role: epel
```

### 7.6.2 核心执行规则

1. **执行顺序**：Ansible 执行角色时，会先按`dependencies`列表的从上到下顺序，依次执行所有依赖角色，再执行当前角色的主任务；
2. **去重规则**：同一个角色被多个角色依赖，默认仅执行一次，如需多次执行，需在依赖角色的`meta/main.yml`中设置`allow_duplicates: yes`；
3. **变量传递**：依赖定义中可给依赖角色传递变量，覆盖其默认值，变量优先级高于角色 defaults，低于 Inventory 变量；
4. **递归依赖**：支持递归依赖，依赖角色自身的`dependencies`也会被自动加载并执行，Ansible 会自动处理依赖树，避免循环依赖；
5. **Play 级继承**：依赖角色会自动继承当前 Play 的`become`、`hosts`等全局配置，无需单独定义。

### 7.6.3 最佳实践与避坑指南

1. **避免循环依赖**：绝对禁止出现 A 依赖 B、B 依赖 A 的循环依赖，会导致 Playbook 解析失败，设计依赖关系时需保持单向依赖；
2. **避免过度依赖**：依赖层级建议不超过 2 层，过度嵌套的依赖会导致执行逻辑混乱、调试困难、维护成本上升；
3. **版本锁定**：依赖第三方角色时，必须锁定版本号，避免依赖角色更新导致的兼容性问题；
4. **依赖职责单一**：仅定义核心强依赖，非必须的依赖不建议定义，由用户在 Playbook 中按需调用；
5. **循环依赖检测**：执行 Playbook 时，可通过`--list-tasks`查看完整的角色执行顺序，验证依赖关系是否符合预期；
6. **禁止硬编码依赖**：避免在角色的任务中通过`import_role`硬编码依赖，统一通过`meta/main.yml`的`dependencies`定义，便于统一管理。

---

## 本章小结

本章完整覆盖了 Ansible Roles 角色体系的全链路内容，严格遵循大纲顺序，从角色的核心设计价值、标准化目录结构，到角色的创建与全场景调用方式、Import 与 Include 的核心底层差异，再到 Ansible Galaxy 社区生态、角色依赖管理，无核心知识点遗漏。

本章内容是 Ansible 从单剧本脚本化到企业级工程化的核心转折点，解决了代码复用、团队协作、标准化、可维护性四大核心工程化问题，是 Ansible 企业级落地的必备核心技能。掌握本章内容，即可构建标准化、可复用、可移植的 Ansible 自动化体系，同时利用全球开源生态，大幅提升自动化开发效率。