## 一、核心定位

`mkdir` 是 Linux 系统中**最核心、最基础的目录创建命令**，属于「02 - 文件目录与路径导航类」，普通用户可在自己有权限的目录下创建新目录。核心作用是**创建单个 / 多个目录，支持递归创建多级目录、创建时直接设置权限、显示详细创建过程等场景**，是文件整理、目录结构搭建、脚本初始化环境的必备基础命令，也是 Linux 入门必学的文件操作命令。

## 二、语法格式

```bash
# 核心语法1：创建单个目录
mkdir [可选参数] 目录名

# 核心语法2：同时创建多个目录
mkdir [可选参数] 目录名1 目录名2 目录名3 ...
```

> 关键说明：
> 
> 1. 无参数时，仅能创建**单级目录**，若父目录不存在会报错；
> 2. 创建**多级目录**（如`a/b/c`）时，**必须加`-p`参数**，否则会因父目录不存在而失败；
> 3. 目录名支持通配符，但需谨慎使用，避免误创建大量目录。

## 三、高频核心参数

| 参数                  | 核心作用                                                    | 高频使用场景                      |
| :------------------ | :------------------------------------------------------ | :-------------------------- |
| `-p, --parents`     | 递归创建多级目录，若父目录不存在则自动创建，已存在的目录不会报错                        | 创建深层目录结构、脚本初始化环境，最核心、最常用的参数 |
| `-m, --mode=MODE`   | 创建目录时**直接设置权限**，无需创建后再用`chmod`修改，权限格式支持数字（如 755、644）和符号 | 批量创建特定权限的目录、安全目录初始化         |
| `-v, --verbose`     | 显示详细的创建过程，输出每个被创建的目录                                    | 监控创建进度、调试目录结构、确认创建内容        |
| `-Z, --context=CTX` | 创建目录时设置 SELinux 安全上下文（仅 SELinux 启用的系统有效）                | SELinux 环境下的目录权限配置          |
| `--help`            | 输出命令帮助信息                                                | 快速查看参数用法                    |
| `--version`         | 输出命令版本信息                                                | 确认命令兼容性                     |

## 四、实操示例

### 1. 基础入门示例（新手必练，零门槛上手）

```bash
# 示例1：最基础用法，创建单个目录
mkdir linux_practice
# 执行效果：在当前目录下创建linux_practice目录
```

```bash
# 示例2：同时创建多个目录
mkdir doc log script
# 执行效果：同时创建doc、log、script三个目录
```

```bash
# 示例3：递归创建多级目录（必须加-p参数，新手必加）
# 创建a/b/c/d深层目录，父目录a、a/b、a/b/c不存在时自动创建
mkdir -p a/b/c/d
# 执行效果：一次性创建完整的多级目录结构，不会报错
```

```bash
# 示例4：显示详细的创建过程
mkdir -pv project/{src,doc,log,backup}
# 执行效果示例：
# mkdir: created directory 'project'
# mkdir: created directory 'project/src'
# mkdir: created directory 'project/doc'
# mkdir: created directory 'project/log'
# mkdir: created directory 'project/backup'
```

---

### 2. 高频日常场景示例

#### 场景 1：创建目录时直接设置权限

```bash
# 创建权限为700的私密目录（仅当前用户可读写执行）
mkdir -m 700 private_dir

# 创建权限为755的公共目录（所有用户可读可执行，仅所有者可写）
mkdir -m 755 public_dir

# 创建权限为777的共享目录（所有用户可读写执行，生产环境谨慎使用）
mkdir -m 777 shared_dir
```

#### 场景 2：批量创建带序号的目录

```bash
# 方式1：结合大括号扩展，批量创建带序号的目录
mkdir -p backup_{01..10}
# 执行效果：创建backup_01到backup_10共10个目录

# 方式2：结合循环，批量创建带日期的目录
for i in {20260301..20260310}; do
    mkdir -p log_$i
done
```

#### 场景 3：脚本中安全创建目录，避免重复创建报错

```bash
#!/bin/bash
# 脚本中安全创建目录的标准写法
TARGET_DIR="/data/backup/$(date +%F)"

# 方式1：先判断目录是否存在，不存在则创建
if [ ! -d "$TARGET_DIR" ]; then
    mkdir -p "$TARGET_DIR"
    echo "目录 $TARGET_DIR 创建成功"
else
    echo "目录 $TARGET_DIR 已存在，跳过创建"
fi

# 方式2：直接用-p创建，已存在的目录不会报错（更简洁）
mkdir -p "$TARGET_DIR"
echo "目录 $TARGET_DIR 已就绪"
```

#### 场景 4：创建带时间戳的目录，用于备份 / 归档

```bash
# 创建格式为backup_YYYYMMDD_HHMMSS的目录
mkdir -p backup_$(date +%Y%m%d_%H%M%S)

# 创建格式为archive_年-月-日的目录
mkdir -p archive_$(date +%F)
```

---

### 3. 进阶奇妙用法

#### 奇妙用法 1：结合大括号扩展，一键创建复杂的项目目录结构

```bash
# 一键创建完整的Web项目目录结构
mkdir -pv web_project/{src/{main/{java,resources,webapp},test/{java,resources}},doc,log,backup,config}
# 执行效果：一次性创建完整的多层级项目目录，无需逐个创建
```

#### 奇妙用法 2：创建临时目录，脚本执行完自动清理

```bash
#!/bin/bash
# 创建临时目录，脚本退出时自动清理
# 方式1：用mktemp创建系统管理的临时目录（推荐，更安全）
TEMP_DIR=$(mktemp -d -t myscript_XXXXXX)
echo "临时目录已创建：$TEMP_DIR"

# 绑定EXIT信号，脚本退出时自动删除临时目录
cleanup() {
    echo "正在清理临时目录..."
    rm -rf "$TEMP_DIR"
}
trap cleanup EXIT

# 模拟脚本业务逻辑
echo "正在临时目录中执行操作..."
touch "$TEMP_DIR/test.txt"
sleep 2

# 脚本结束时，cleanup函数会自动执行，删除临时目录
echo "脚本执行完成"
```

#### 奇妙用法 3：结合 find，查找空目录并批量创建子目录

```bash
# 查找当前目录下所有空目录，在每个空目录中创建一个sub_dir子目录
find . -type d -empty -exec mkdir -p {}/sub_dir \;
```

#### 奇妙用法 4：创建带空格 / 特殊字符的目录

```bash
# 目录名带空格，必须用双引号包裹
mkdir "My Project"

# 目录名带特殊字符，用双引号包裹或反斜杠转义
mkdir "Project&Backup"
# 或
mkdir Project\&Backup
```

#### 奇妙用法 5：批量创建用户专属目录

```bash
# 假设用户列表为user1、user2、user3，批量创建每个用户的专属目录并设置权限
for user in user1 user2 user3; do
    mkdir -p /data/home/$user
    chown $user:$user /data/home/$user
    chmod 700 /data/home/$user
done
```

## 五、避坑提示 & 注意事项

1. **新手最高频致命坑：创建多级目录忘记加`-p`**
    
    - 创建多级目录（如`a/b/c`）时，**必须加`-p`参数**，否则会因父目录不存在而报错`mkdir: cannot create directory 'a/b/c': No such file or directory`；
    - 强制要求：创建目录时，**优先加`-p`参数**，即使是单级目录也不会有副作用，还能避免父目录不存在的报错。
    
2. **权限不足的坑**
    
    - 在无权限的目录（如`/root`、`/etc`、`/data`）下创建目录，会报错`mkdir: cannot create directory 'xxx': Permission denied`；
    - 解决方法：在系统目录下创建需加`sudo`，或在自己有权限的目录（如家目录、`/tmp`）下创建。
    
3. **目录名带空格 / 特殊字符的坑**
    
    - 目录名包含空格、`&`、`!`、`()`、`;`等特殊字符时，**必须用双引号包裹**，或用反斜杠转义；
    - 错误示例：`mkdir My Project`（会被识别为两个目录，创建 My 和 Project）；
    - 正确示例：`mkdir "My Project"` 或 `mkdir My\ Project`。
    
4. **覆盖已存在目录的坑**
    
    - `mkdir` 默认不会覆盖已存在的目录，会报错`mkdir: cannot create directory 'xxx': File exists`；
    - 注意：`mkdir` 没有`-f`（强制）参数，不要和`rm`、`cp`混淆；
    - 若需确保目录存在且不报错，直接加`-p`参数即可，已存在的目录不会被修改。
    
5. **目录名的命名规范**
    
    - 目录名不能包含`/`（路径分隔符）、`\0`（空字符）；
    - 避免使用`-`开头的目录名，会被误认为是参数，若必须使用，需用`--`分隔，如`mkdir -- -my-dir`；
    - 建议使用小写字母、数字、下划线`_`、连字符`-`，避免空格和特殊字符，方便脚本处理。
    
6. **SELinux 上下文的坑**
    
    - SELinux 启用的系统中，创建的目录会继承父目录的 SELinux 上下文；
    - 若需自定义 SELinux 上下文，需加`-Z`参数，如`mkdir -Z system_u:object_r:httpd_sys_content_t:s0 /var/www/html`。
    
7. **临时目录的安全清理**
    
    - 脚本中创建临时目录时，必须绑定`EXIT`信号，确保脚本退出时自动清理，避免临时文件堆积；
    - 优先使用`mktemp -d`创建临时目录，系统会自动管理权限和位置，比手动在`/tmp`下创建更安全。
    

## 六、同类命令对比 & 拓展

表格

|命令|核心作用|与 mkdir 的区别 & 适用场景|
|:--|:--|:--|
|`rmdir`|删除空目录|mkdir 创建目录，rmdir 删除空目录；两者互补，创建用 mkdir，删除空目录用 rmdir|
|`touch`|创建空文件或修改文件时间戳|mkdir 创建目录，touch 创建文件；两者是文件系统操作的黄金搭档，目录用 mkdir，文件用 touch|
|`install`|复制文件并设置权限，也可创建目录|install 可同时创建目录、复制文件、设置权限，适合软件安装脚本；mkdir 专注于目录创建，更通用|
|`mktemp`|创建临时目录 / 文件|mktemp 创建系统管理的临时目录 / 文件，自动清理更安全；mkdir 创建永久目录，临时目录优先用 mktemp -d|
|`cp -r`|复制目录|cp -r 复制已存在的目录结构，mkdir 创建新目录；复制目录用 cp -r，创建新目录用 mkdir|
|`rsync`|远程 / 本地目录同步|rsync 可同步目录结构，不存在的目录会自动创建；mkdir 专注于本地创建，远程同步用 rsync|

## 七、课后练习任务

1. 基础创建练习：在当前目录下创建`test_dir`目录，同时创建`dir1`、`dir2`、`dir3`三个目录。
2. 多级目录创建练习：用`mkdir -p`创建`project/src/main/java`、`project/src/test/java`、`project/doc`、`project/log`完整的项目目录结构，同时显示详细创建过程。
3. 权限设置练习：创建`private`目录，权限设置为 700；创建`public`目录，权限设置为 755；用`ls -ld`验证权限是否正确。
4. 批量创建练习：用大括号扩展批量创建`backup_20260301`到`backup_20260310`共 10 个目录。
5. 带时间戳目录创建练习：创建格式为`data_YYYYMMDD_HHMMSS`的目录，记录创建的目录名。
6. 临时目录创建练习：写一个脚本，用`mktemp -d`创建临时目录，绑定 EXIT 信号，脚本退出时自动删除临时目录，在临时目录中创建一个测试文件，验证脚本执行完后临时目录是否被清理。
7. 复杂目录结构创建练习：结合大括号扩展，一键创建`webapp/{static/{css,js,img},templates,config,log,backup}`完整的 Web 应用目录结构。