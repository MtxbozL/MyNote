`mkdir` 是 Linux/Unix 类系统中**最核心、最基础的目录操作命令**，全称为 `make directories`，用于创建新的目录（文件夹），是 shell 操作、脚本编写、运维工作的必备命令，遵循 POSIX 标准，兼容所有 Linux、macOS、BSD 等类 Unix 系统。

---

## 一、命令基础信息

### 1. 核心功能

- 核心作用：创建一个或多个空目录
- 补充能力：递归创建多级目录、创建时直接指定目录权限、显示创建详情、设置安全上下文等
- 执行前提：在目标父目录中必须拥有**写权限（w）**和**执行权限（x）**，否则会创建失败

### 2. 标准语法格式

bash

运行

```
mkdir [选项] 目录名/路径
```

- `[选项]`：可选参数，用于扩展命令功能，多个选项可组合使用
- `目录名/路径`：必填参数，支持相对路径、绝对路径，可同时指定多个目标

---

## 二、基础入门用法（零选项必学）

无任何选项时，`mkdir` 仅能创建**同级单层目录**，无法创建多级目录，若目标目录已存在会直接报错。

### 1. 创建单个目录

bash

运行

```
# 语法：mkdir 目录名
mkdir test_dir
```

- 示例说明：在当前工作目录下，创建一个名为 `test_dir` 的空目录
- 补充：默认创建的目录权限由系统 `umask` 值决定（默认 umask 022 对应目录权限 755）

### 2. 同时创建多个同级目录

bash

运行

```
# 语法：mkdir 目录1 目录2 目录3 ...
mkdir dir1 dir2 dir3 doc img data
```

- 示例说明：在当前目录下，一次性创建 `dir1`、`dir2`、`dir3`、`doc`、`img`、`data` 6 个同级目录

### 3. 按绝对 / 相对路径创建目录

bash

运行

```
# 相对路径：在上级目录创建目录
mkdir ../parent_dir

# 绝对路径：在指定系统路径创建目录
mkdir /home/user/workspace/project
mkdir /var/log/nginx_log
```

- 注意：绝对路径创建时，必须保证路径中**除最后一级目录外，前面的父目录都已存在**，否则会报错。

---

## 三、核心可选参数（选项）详解

### 高频必学选项（日常使用 90% 场景覆盖）

表格

|简写选项|全称选项|核心功能|
|---|---|---|
|`-p`|`--parents`|递归创建多级目录，父目录不存在自动创建；目录已存在时不报错，无覆盖风险|
|`-m`|`--mode`|创建目录时直接指定权限，无需后续用 chmod 二次修改|
|`-v`|`--verbose`|显示目录创建的详细过程，输出每一步操作结果，用于调试和确认|

#### 1. `-p` 递归创建多级目录（最常用）

这是 `mkdir` 最高频的选项，解决了无选项时无法创建多级目录的痛点。

bash

运行

```
# 错误示例：无-p时，父目录不存在会直接报错
mkdir a/b/c/d
# 报错：mkdir: cannot create directory ‘a/b/c/d’: No such file or directory

# 正确示例：递归创建多级目录
mkdir -p a/b/c/d
```

- 核心特性 1：自动补全路径中所有不存在的父目录，无需逐层创建
- 核心特性 2：若目标目录已存在，不会抛出 `File exists` 错误，也不会修改原有目录的内容和权限，安全无风险
- 进阶用法：同时递归创建多组多级目录

bash

运行

```
mkdir -p project/{src,doc,conf,log} project/src/{utils,models,api}
```

执行后会自动生成以下完整目录树：

plaintext

```
project/
├── src/
│   ├── utils/
│   ├── models/
│   └── api/
├── doc/
├── conf/
└── log/
```

#### 2. `-m` 创建时直接指定目录权限

跳过 `chmod` 二次修改，创建时直接锁定目录权限，适用于生产环境权限管控场景。

bash

运行

```
# 语法：mkdir -m 权限数值 目录名
mkdir -m 700 private_dir
mkdir -m 755 public_dir
mkdir -m 777 share_dir
```

- 权限说明：采用 Linux 八进制权限数值（用户 - 组 - 其他），目录建议使用 7 开头的权限（目录必须有执行权限 x 才能进入）
- 补充：该权限会直接覆盖系统 `umask` 的默认规则，无需考虑 umask 影响

#### 3. `-v` 显示创建详细过程

清晰输出每一个目录的创建结果，避免盲操作，适合脚本调试、批量创建确认场景。

bash

运行

```
mkdir -v dir1 dir2 dir3
# 输出：
# mkdir: created directory 'dir1'
# mkdir: created directory 'dir2'
# mkdir: created directory 'dir3'

# 组合-p使用，查看递归创建的完整过程
mkdir -pv a/b/c/d
# 输出：
# mkdir: created directory 'a'
# mkdir: created directory 'a/b'
# mkdir: created directory 'a/b/c'
# mkdir: created directory 'a/b/c/d'
```

### 进阶实用选项

表格

|简写选项|全称选项|核心功能|
|---|---|---|
|`-Z`|`--context`|创建目录时同时设置 SELinux 安全上下文，仅适用于开启 SELinux 的系统（RHEL/CentOS）|
|无|`--help`|查看 mkdir 命令的完整帮助文档，退出后返回终端|
|无|`--version`|查看 mkdir 命令的版本信息（属于 GNU coreutils 工具集）|

#### 示例：`-Z` 设置 SELinux 上下文

bash

运行

```
# 为web目录设置httpd可访问的SELinux上下文
mkdir -Z system_u:object_r:httpd_sys_content_t:s0 /var/www/html/web_app
```

---

## 四、进阶实战用法

### 1. 批量创建有序目录（花括号扩展）

结合 shell 花括号扩展语法，一键批量创建有规律的目录，无需循环脚本，是运维和测试的高频技巧。

bash

运行

```
# 批量创建有序数字目录：dir01 到 dir10
mkdir dir{01..10}

# 批量创建带前缀的分类目录
mkdir log_{2025,2026}_{01..12}

# 组合-p批量创建多级分类目录
mkdir -p project/{dev,test,prod}/{log,data,conf}
```

### 2. 带特殊字符的目录创建

当目录名包含空格、横杠、特殊符号时，需用引号包裹或反斜杠转义，避免被 shell 解析为命令参数。

bash

运行

```
# 1. 带空格的目录名：双引号/单引号包裹
mkdir "my project"
mkdir 'work space'

# 2. 带空格的目录名：反斜杠转义
mkdir my\ project

# 3. 以横杠-开头的目录名（避免被识别为选项）
mkdir -- -test_dir
mkdir ./-test_dir

# 4. 带特殊符号的目录名
mkdir "dir_with_@#$"
```

### 3. 生产级标准化目录一键创建

日常开发 / 运维中，可通过一条命令创建符合规范的项目目录结构，无需手动逐层创建。

bash

运行

```
# 示例：一键创建Python后端项目标准目录结构
mkdir -pv my_python_project/{
    src/{core,api,utils,models,middleware},
    tests/{unit,integraion},
    docs/{api,deploy},
    data/{raw,processed},
    logs,
    conf,
    scripts,
    static
}
```

---

## 五、避坑指南与注意事项

### 1. 高频报错与解决方案

表格

|报错信息|报错原因|解决方案|
|---|---|---|
|`cannot create directory 'xxx': No such file or directory`|路径中父目录不存在，未加`-p`选项|补充`-p`选项递归创建，或先创建父目录|
|`cannot create directory 'xxx': File exists`|目标目录已存在，重复执行创建命令|加`-p`选项（已存在目录不报错），或先删除旧目录|
|`cannot create directory 'xxx': Permission denied`|当前用户在父目录无写 / 执行权限|切换到有权限的目录，或用 sudo 提升权限，或修改父目录权限|
|`invalid mode`|`-m`选项指定的权限格式错误|使用合法的八进制权限数值（如 755、700），勿用字母权限|

### 2. 关键注意事项

1. **目录覆盖风险**：`mkdir` 永远不会覆盖已存在的目录，即使加`-p`选项，也仅会跳过已存在目录，不会修改其内容和权限，无数据丢失风险。
2. **权限前提**：创建目录的核心前提是，对目标路径的**父目录拥有 w（写）和 x（执行）权限**，root 用户不受此限制。
3. **umask 影响**：无`-m`选项时，目录默认权限 = 777 - umask 值。例如系统默认 umask=022，默认目录权限为 755；umask=002，默认权限为 775。
4. **特殊路径规范**：创建系统级目录（如 /var、/etc、/usr 下）时，必须使用 sudo 提升权限，避免权限不足报错。

---

## 六、配套关联命令

表格

|命令|核心关联功能|
|---|---|
|`rmdir`|删除空目录，与 mkdir 成对使用，仅能删除空目录|
|`rm -r`|递归删除目录（含非空目录），高危操作，谨慎使用|
|`cd`|切换到创建好的目录，配合 mkdir 使用|
|`ls -ld`|查看目录的权限、属性、创建时间等信息|
|`chmod`|修改目录权限，与 mkdir -m 功能互补|
|`chown`|修改目录的所属用户和用户组|
|`tree`|查看创建的目录树结构，验证创建结果|

---

## 七、综合实战练习

1. 在家目录下，创建一个名为 `my_work` 的目录，并查看创建结果
2. 一键递归创建 `project/src/main/java` 多级目录，要求显示完整创建过程
3. 批量创建 `log_01` 到 `log_12` 共 12 个日志目录
4. 创建一个只有所属用户可读写执行（权限 700）的私密目录 `private_data`
5. 在家目录下，创建一个名为 `my project` 带空格的目录，并成功进入该目录