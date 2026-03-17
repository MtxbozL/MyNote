## 一、核心定位

`touch` 是 Linux 系统中**创建空文件与修改文件时间戳的核心命令**，属于「02 - 文件目录与路径导航类」，普通用户可在自己有权限的目录下操作。核心作用是**创建空文件、更新文件的访问时间（atime）、修改时间（mtime）、元数据修改时间（ctime），支持指定时间、参考其他文件时间、批量创建等场景**，是测试文件创建、脚本初始化、时间戳调整、文件存在性标记的必备基础命令，也是 Linux 入门必学的文件操作命令。

> 补充说明：`touch` 的主要功能是修改时间戳，创建空文件是其附带功能；若仅需创建空文件，`touch` 是最简洁的方式。

## 二、语法格式

```bash
# 核心语法1：创建空文件/更新文件时间戳
touch [可选参数] 文件1 文件2 文件3 ...
```

> 关键说明：
> 
> 1. 若文件不存在：默认创建一个**空文件**（大小为 0 字节）；
> 2. 若文件已存在：默认同时更新文件的**访问时间（atime）和修改时间（mtime）为当前系统时间**；
> 3. 元数据修改时间（ctime）无法直接指定，只要文件的 atime/mtime/ 权限 / 所有者等发生变化，ctime 会自动更新为当前时间。

## 三、高频核心参数

|参数|核心作用|高频使用场景|
|:--|:--|:--|
|无参数（直接执行`touch`）|文件不存在则创建空文件，文件已存在则同时更新 atime 和 mtime 为当前时间|日常快速创建空文件、更新文件时间戳，最核心、最常用的用法|
|`-a`|仅更新文件的**访问时间（atime）**，不更新修改时间|仅需修改访问时间的场景|
|`-m`|仅更新文件的**修改时间（mtime）**，不更新访问时间|仅需修改修改时间的场景，日常最常用的时间戳更新方式|
|`-c, --no-create`|不创建新文件，仅更新已存在文件的时间戳|避免误创建不存在的文件，仅调整已有文件的时间戳|
|`-d, --date=STRING`|指定时间字符串，而非当前时间，支持多种自然语言格式（如 "2026-03-15"、"yesterday"、"2 days ago"、"10:30"）|灵活指定任意时间，时间戳调整首选|
|`-t STAMP`|指定时间戳，格式为`[[CC]YY]MMDDhhmm[.ss]`（如`202603151030`代表 2026 年 3 月 15 日 10:30）|精确指定时间，脚本中时间戳标准化|
|`-r, --reference=FILE`|参考指定文件的时间戳，将目标文件的时间戳更新为与参考文件一致|批量同步文件时间戳、保持文件时间一致性|
|`--help`|输出命令帮助信息|快速查看参数用法|
|`--version`|输出命令版本信息|确认命令兼容性|

## 四、实操示例

### 1. 基础入门示例（新手必练，零门槛上手）

```bash
# 示例1：最基础用法，创建单个空文件
touch test.txt
# 执行效果：若test.txt不存在则创建空文件，若已存在则更新其atime和mtime为当前时间
```

```bash
# 示例2：同时创建多个空文件
touch file1.txt file2.txt file3.txt
# 或用大括号扩展批量创建
touch file_{01..10}.txt
# 执行效果：同时创建多个空文件
```

```bash
# 示例3：仅更新文件的修改时间（mtime）
touch -m test.txt
# 执行效果：仅更新test.txt的修改时间为当前时间，访问时间不变
```

```bash
# 示例4：仅更新文件的访问时间（atime）
touch -a test.txt
# 执行效果：仅更新test.txt的访问时间为当前时间，修改时间不变
```

```bash
# 示例5：不创建新文件，仅更新已存在文件的时间戳
touch -c non_existent_file.txt
# 执行效果：若non_existent_file.txt不存在，不会创建，仅跳过
```

---

### 2. 高频日常场景示例

#### 场景 1：指定任意时间更新文件时间戳

```bash
# 方式1：用-d参数指定自然语言时间
# 更新为指定日期
touch -d "2026-03-15" test.txt
# 更新为昨天
touch -d "yesterday" test.txt
# 更新为2天前
touch -d "2 days ago" test.txt
# 更新为指定时间
touch -d "10:30" test.txt
# 更新为指定日期时间
touch -d "2026-03-15 10:30:45" test.txt
```

```bash
# 方式2：用-t参数指定精确时间戳
# 格式：[[CC]YY]MMDDhhmm[.ss]
touch -t 202603151030 test.txt
# 带秒数
touch -t 202603151030.45 test.txt
```

#### 场景 2：参考其他文件的时间戳

```bash
# 创建一个参考文件
touch -d "2026-01-01" reference.txt

# 将test.txt的时间戳更新为与reference.txt一致
touch -r reference.txt test.txt
# 执行效果：test.txt的atime和mtime与reference.txt完全一致
```

#### 场景 3：批量创建带序号的测试文件

```bash
# 批量创建file_001.txt到file_100.txt共100个测试文件
touch file_{001..100}.txt
```

#### 场景 4：标记文件存在性，用于脚本判断

```bash
#!/bin/bash
# 用touch创建标记文件，用于脚本判断是否已执行过
MARK_FILE="/tmp/script_executed.mark"

# 判断标记文件是否存在
if [ -f "$MARK_FILE" ]; then
    echo "脚本已执行过，跳过"
    exit 0
fi

# 执行核心逻辑
echo "脚本第一次执行，开始执行任务..."
# 业务逻辑
sleep 2

# 执行完成后创建标记文件
touch "$MARK_FILE"
echo "任务执行完成，标记文件已创建"
```

---

### 3. 进阶奇妙用法

#### 奇妙用法 1：批量更新目录下所有文件的时间戳

```bash
# 批量更新当前目录下所有.txt文件的修改时间为当前时间
touch -m *.txt

# 批量更新当前目录及其子目录下所有文件的时间戳
find . -type f -exec touch {} \;
```

#### 奇妙用法 2：创建带时间戳的文件

```bash
# 创建格式为file_YYYYMMDD_HHMMSS.txt的文件
touch "file_$(date +%Y%m%d_%H%M%S).txt"
```

#### 奇妙用法 3：结合 find，更新特定条件文件的时间戳

```bash
# 查找所有30天前的.log文件，将其时间戳更新为当前时间
find /var/log -type f -name "*.log" -mtime +30 -exec touch {} \;

# 查找所有大于100M的文件，将其时间戳更新为7天前
find /data -type f -size +100M -exec touch -d "7 days ago" {} \;
```

#### 奇妙用法 4：模拟文件修改，触发监控系统

```bash
# 定期更新文件时间戳，模拟文件修改，触发监控系统的告警
while true; do
    touch -m /data/monitor_file.txt
    echo "文件时间戳已更新：$(date)"
    sleep 60  # 每分钟更新一次
done
```

#### 奇妙用法 5：批量同步文件时间戳

```bash
# 将源目录下所有文件的时间戳同步到目标目录
SOURCE_DIR="/data/source"
TARGET_DIR="/data/target"

# 遍历源目录下所有文件
for file in "$SOURCE_DIR"/*; do
    if [ -f "$file" ]; then
        filename=$(basename "$file")
        # 若目标文件存在，同步时间戳
        if [ -f "$TARGET_DIR/$filename" ]; then
            touch -r "$file" "$TARGET_DIR/$filename"
            echo "已同步 $filename 的时间戳"
        fi
    fi
done
```

## 五、避坑提示 & 注意事项

1. **新手高频踩坑：touch 默认会同时更新 atime 和 mtime**
    
    - 无参数时，`touch` 会同时更新文件的**访问时间（atime）和修改时间（mtime）**；
    - 若仅需更新其中一个，必须加`-a`（仅 atime）或`-m`（仅 mtime）参数；
    - 注意：元数据修改时间（ctime）无法直接指定，只要 atime/mtime 发生变化，ctime 会自动更新为当前时间。
    
2. **`-c`参数的作用**
    
    - `-c` 参数仅更新已存在文件的时间戳，**不会创建新文件**；
    - 若需确保不创建不存在的文件，必须加`-c`参数。
    
3. **时间格式的注意事项**
    
    - `-d` 参数支持多种自然语言格式，但不同系统的支持程度可能存在差异；
    - `-t` 参数的格式必须严格遵循`[[CC]YY]MMDDhhmm[.ss]`，否则会报错；
    - 脚本中为保证兼容性，建议使用`-t`参数的标准格式。
    
4. **权限不足的坑**
    
    - 修改系统文件（如`/root`、`/etc`、`/var`下的文件）的时间戳时，普通用户会报错`touch: cannot touch 'xxx': Permission denied`；
    - 解决方法：修改系统文件需加`sudo`。
    
5. **符号链接的处理坑**
    
    - `touch` 默认修改符号链接本身的时间戳，而非符号链接指向的目标文件；
    - 若要修改符号链接指向的目标文件的时间戳，需加`-h`参数（部分系统支持），或先跟随符号链接找到目标文件，再修改。
    
6. **空文件的大小**
    
    - `touch` 创建的空文件大小为**0 字节**，不占用磁盘数据块（但会占用 inode）；
    - 若需创建非空文件，需用`echo`、`cat`、`>`等命令写入内容。
    
7. **文件系统的 atime 挂载选项**
    
    - 若文件系统以`noatime`或`relatime`选项挂载，`touch -a`可能无法正常更新访问时间；
    - 解决方法：查看文件系统挂载选项`mount | grep noatime`，若需更新 atime，需重新挂载文件系统。
    

## 六、同类命令对比 & 拓展

表格

|命令|核心作用|与 touch 的区别 & 适用场景|
|:--|:--|:--|
|`mkdir`|创建目录|touch 创建空文件，mkdir 创建目录；两者是文件系统操作的黄金搭档，文件用 touch，目录用 mkdir|
|`cp`|复制文件|touch 创建空文件或修改时间戳，cp 复制已存在的文件；创建空文件用 touch，复制文件用 cp|
|`> filename`|重定向创建空文件或覆盖文件|若文件不存在则创建空文件，若已存在则覆盖；touch 仅创建空文件或更新时间戳，不会覆盖文件内容，安全创建空文件优先用 touch|
|`>> filename`|重定向追加内容到文件|若文件不存在则创建空文件并追加内容，若已存在则追加内容；touch 仅创建空文件，创建带内容的文件用 >>|
|`stat`|查看文件的详细元数据|可查看文件的 atime、mtime、ctime 三种时间戳的详细信息（精确到纳秒）；touch 用于修改时间戳，stat 用于查看时间戳，两者互补|
|`date`|显示 / 设置系统时间|用于查看或修改系统时间；touch 用于修改文件时间戳，系统时间用 date，文件时间戳用 touch|
|`utime`|修改文件时间戳的系统调用|是 touch 的底层系统调用，功能更底层；日常使用优先用 touch，底层开发用 utime|

## 七、课后练习任务

1. 基础创建练习：在当前目录下创建`test.txt`空文件，同时创建`file1.txt`、`file2.txt`、`file3.txt`三个空文件。
2. 批量创建练习：用大括号扩展批量创建`file_001.txt`到`file_010.txt`共 10 个空文件。
3. 时间戳更新练习：创建`test.txt`文件，分别用`touch -a`、`touch -m`、`touch`三种方式更新时间戳，用`stat`命令查看三种时间戳的变化。
4. 指定时间练习：用`touch -d`将`test.txt`的时间戳更新为 “2026-01-01”，用`touch -t`将其更新为 “202603151030”，用`stat`验证。
5. 参考文件练习：创建`reference.txt`文件，将其时间戳更新为 “yesterday”，然后用`touch -r`将`test.txt`的时间戳同步为与`reference.txt`一致，验证结果。
6. 不创建文件练习：尝试用`touch -c`更新一个不存在的文件，验证是否会创建新文件。
7. 结合 find 练习：用`find`查找当前目录下所有`.txt`文件，批量将它们的时间戳更新为 “2 days ago”。