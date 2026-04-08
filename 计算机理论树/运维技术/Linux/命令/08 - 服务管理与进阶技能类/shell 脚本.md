![[Pasted image 20260310223106.png]]

export 命令

本笔记基于**Bash（Bourne-Again Shell）** 编写（Linux 系统默认 Shell），覆盖从零基础入门到企业级实战的全链路知识点，兼顾语法详解、实操案例、避坑指南和最佳实践，可作为学习手册与日常开发查阅工具书。

---

## 一、Shell 基础入门

### 1.1 核心概念

- **Shell**：是操作系统的用户与内核之间的命令解释器，负责接收用户输入的命令，翻译给内核执行，并将执行结果返回给用户。
- **常见 Shell 类型**：

|Shell 类型|说明|
|---|---|
|Bash|绝大多数 Linux 发行版默认，功能最全，兼容性最好|
|Sh|极简 POSIX 标准 Shell，很多 Linux 系统中 sh 是 bash 的软链接|
|Zsh|兼容 bash，支持更多插件和个性化配置，常用搭配 oh-my-zsh|
|Dash|Debian/Ubuntu 默认 sh，轻量高效，仅支持 POSIX 标准|

- **脚本本质**：将一系列 Shell 命令按逻辑写入文件，批量、自动化执行，避免重复手动输入命令。

### 1.2 第一个 Shell 脚本

#### 1.2.1 脚本创建

```bash
# 1. 创建脚本文件
vim hello.sh
```

#### 1.2.2 脚本内容

```bash
#!/bin/bash
# 我的第一个Shell脚本
# 井号开头（除#!外）均为注释，不会被执行

# 输出内容到控制台
echo "Hello Shell World!"
```

#### 1.2.3 核心说明

- **`#!/bin/bash`**：称为`Shebang`，必须放在脚本**第一行**，声明脚本使用的解释器为 /bin/bash，系统会根据该声明调用对应解释器执行脚本。
- 若不写 Shebang，系统会默认用当前 Shell 执行脚本，可能出现兼容性问题。

### 1.3 脚本的执行方式与核心区别

表格

|执行方式|命令示例|执行原理|权限要求|执行环境|
|---|---|---|---|---|
|路径执行（推荐）|`./hello.sh`|系统读取 Shebang，启动对应解释器执行|必须有**执行权限**（x）|开启**子 Shell**进程执行，脚本内变量不影响当前终端|
|解释器直接调用|`bash hello.sh` / `sh hello.sh`|直接启动 bash/sh 程序，将脚本作为参数传入|无需执行权限，只需读权限|开启**子 Shell**进程执行，脚本内变量不影响当前终端|
|source / 点执行|`source hello.sh` / `. hello.sh`|在当前 Shell 环境中逐行执行脚本内容|无需执行权限，只需读权限|**当前 Shell**执行，脚本内变量、环境修改会直接影响当前终端|

#### 权限添加命令

```bash
# 给脚本添加执行权限（仅需执行一次）
chmod +x hello.sh
```

---

## 二、Shell 核心语法基础

### 2.1 变量详解

Shell 变量无需提前声明类型，默认均为字符串类型，赋值时直接`变量名=值`，**等号两侧绝对不能有空格**。

#### 2.1.1 变量命名规则

1. 只能由字母、数字、下划线组成，不能以数字开头
2. 区分大小写
3. 不能使用 Shell 关键字（if、for、while 等）
4. 建议见名知意，全大写命名环境变量，小写命名自定义变量

#### 2.1.2 变量的 4 大类型

##### 1. 自定义局部变量

```bash
#!/bin/bash
# 变量定义
name="张三"
age=20
# 变量引用：$变量名 或 ${变量名}（推荐，避免歧义）
echo "姓名：$name"
echo "年龄：${age}岁"

# 变量重新赋值
name="李四"
echo "修改后的姓名：${name}"
```

##### 2. 只读变量（readonly）

定义后不可修改、不可删除，常用于定义固定常量

```bash
#!/bin/bash
readonly PI=3.1415926
echo "圆周率：${PI}"
# PI=3.14  # 执行会报错，只读变量无法修改
```

##### 3. 环境变量

系统预定义的全局变量，所有 Shell 进程都能访问，常用`export`声明自定义环境变量

```bash
# 查看所有环境变量
env
# 查看单个环境变量
echo $PATH
echo $HOME
echo $USER
echo $PWD

# 自定义环境变量（临时生效，重启终端失效）
export MY_ENV="自定义环境变量"
echo $MY_ENV

# 永久生效：写入/etc/profile（全局）或 ~/.bashrc（当前用户），执行source生效
```

##### 4. 特殊内置变量（高频使用）

Shell 预定义的变量，用于获取脚本参数、执行状态等核心信息，不可修改

表格

|变量|核心含义|
|---|---|
|`$0`|当前脚本的文件名|
|`$1 ~ $n`|脚本 / 函数的第 1~n 个输入参数，n>9 时必须用`${10}`格式|
|`$#`|脚本 / 函数的输入参数总个数|
|`$*`|所有输入参数，不加引号时等同于`$@`；加双引号时`"$*"`将所有参数合并为一个字符串|
|`$@`|所有输入参数，加双引号时`"$@"`将每个参数作为独立字符串（遍历参数推荐）|
|`$?`|上一条命令 / 函数的退出状态码，0 = 执行成功，非 0 = 执行失败（核心，用于判断命令执行结果）|
|`$$`|当前 Shell 进程的 PID|
|`$!`|后台运行的最后一个进程的 PID|

```bash
#!/bin/bash
# 特殊变量示例
echo "脚本文件名：$0"
echo "第1个参数：$1"
echo "第2个参数：$2"
echo "参数总个数：$#"
echo "所有参数：$@"

# 遍历参数（推荐$@）
echo "===== 遍历参数 ====="
for arg in "$@"
do
    echo "参数：${arg}"
done

# 上一条命令执行结果
echo "上一条命令退出码：$?"
```

#### 2.1.3 变量的删除

```bash
name="张三"
echo $name
# 删除变量
unset name
echo $name  # 输出为空
```

### 2.2 字符串处理（Shell 最常用操作）

Shell 中所有数据默认都是字符串，单双引号有本质区别，需重点区分。

#### 2.2.1 单引号 vs 双引号

|类型|特性|适用场景|
|---|---|---|
|单引号`''`|1. 所有字符原样输出，变量、转义符均失效；2. 单引号内不能嵌套单引号|纯静态字符串，无需解析变量|
|双引号`""`|1. 可识别变量、转义符（\n、\t 等）；2. 可嵌套双引号 / 单引号|动态字符串，需要解析变量|

```bash
#!/bin/bash
name="张三"
# 单引号
echo '姓名：$name \n 换行'  # 输出：姓名：$name \n 换行
# 双引号
echo "姓名：$name \n 换行"  # 输出：姓名：张三 （换行） 换行
```

#### 2.2.2 核心字符串操作

```bash
#!/bin/bash
str="hello shell world"

# 1. 获取字符串长度
echo "字符串长度：${#str}"  # 输出：17

# 2. 字符串切片：${str:起始索引:截取长度}，索引从0开始
echo "从第6个字符开始截取：${str:6}"  # 输出：shell world
echo "从第6个字符开始，截取5个：${str:6:5}"  # 输出：shell
echo "从倒数第5个字符开始截取：${str:(-5)}"  # 输出：world

# 3. 字符串替换
# 3.1 替换第一个匹配项
echo "替换第一个l为L：${str/l/L}"  # 输出：heLlo shell world
# 3.2 替换所有匹配项
echo "替换所有l为L：${str//l/L}"  # 输出：heLLo sheLL worLd

# 4. 字符串删除
# 4.1 从开头删除最短匹配
echo "删除开头到第一个空格：${str#* }"  # 输出：shell world
# 4.2 从开头删除最长匹配
echo "删除开头到最后一个空格：${str##* }"  # 输出：world
# 4.3 从结尾删除最短匹配
echo "删除结尾到第一个空格：${str% *}"  # 输出：hello shell
# 4.4 从结尾删除最长匹配
echo "删除结尾到最后一个空格：${str%% *}"  # 输出：hello
```

### 2.3 数组操作

Bash 支持**普通索引数组**和**关联数组**（键值对，Bash 4.0 + 版本支持）。

#### 2.3.1 普通索引数组

```bash
#!/bin/bash
# 1. 数组定义（元素用空格分隔）
arr=("苹果" "香蕉" "橙子" "葡萄" "西瓜")
# 单独赋值
arr[5]="芒果"

# 2. 读取数组元素
echo "第1个元素：${arr[0]}"  # 索引从0开始
echo "第3个元素：${arr[2]}"

# 3. 读取数组所有元素
echo "数组所有元素：${arr[@]}"  # 推荐@，遍历用
echo "数组所有元素：${arr[*]}"

# 4. 获取数组长度
echo "数组元素个数：${#arr[@]}"

# 5. 数组切片
echo "从第2个元素开始，取3个：${arr[@]:1:3}"

# 6. 遍历数组（推荐方式）
echo "===== 遍历数组 ====="
for item in "${arr[@]}"
do
    echo "水果：${item}"
done

# 7. 数组删除
unset arr[2]  # 删除指定索引元素
echo "删除后的数组：${arr[@]}"
unset arr  # 删除整个数组
```

#### 2.3.2 关联数组（键值对）

```bash
#!/bin/bash
# 1. 先声明关联数组
declare -A user_info

# 2. 赋值
user_info["name"]="张三"
user_info["age"]=20
user_info["gender"]="男"

# 3. 读取元素
echo "姓名：${user_info["name"]}"
echo "年龄：${user_info["age"]}"

# 4. 获取所有键/值
echo "所有键：${!user_info[@]}"
echo "所有值：${user_info[@]}"

# 5. 遍历关联数组
echo "===== 遍历关联数组 ====="
for key in "${!user_info[@]}"
do
    echo "${key}：${user_info[$key]}"
done
```

### 2.4 运算符大全

Shell 默认不支持浮点运算，仅支持整数运算，浮点运算需借助`bc`等工具。

#### 2.4.1 算术运算符

|运算符|含义|示例|
|---|---|---|
|`+`|加法|`$((a + b))`|
|`-`|减法|`$((a - b))`|
|`*`|乘法|`$((a * b))`|
|`/`|除法（取整）|`$((a / b))`|
|`%`|取余（模运算）|`$((a % b))`|
|`++`|自增|`$((a++))` / `$((++a))`|
|`--`|自减|`$((a--))` / `$((--a))`|

##### 算术运算的 4 种实现方式

```bash
#!/bin/bash
a=10
b=3

# 1. 推荐方式：$(( )) （Bash内置，无需额外命令，无空格限制）
echo "加法：$((a + b))"
echo "乘法：$((a * b))"
echo "取余：$((a % b))"
# 复合赋值
((a += 5))
echo "a自增5后：${a}"

# 2. let命令（内置，适合简单赋值）
let "c = a + b"
echo "c = ${c}"

# 3. expr命令（POSIX兼容，运算符两侧必须有空格，*需要转义）
d=`expr $a + $b`
echo "d = ${d}"
e=`expr $a \* $b`
echo "e = ${e}"

# 4. 浮点运算：bc工具（支持小数，scale指定小数位数）
echo "10/3 保留2位小数：`echo "scale=2; 10/3" | bc`"
echo "3.14*2.5：`echo "3.14*2.5" | bc`"
```

#### 2.4.2 关系运算符（仅用于数值比较）

用于两个数值的比较，返回布尔值，**仅支持整数**，配合`[ ]`/`[[ ]]`使用

|运算符|含义|示例|
|---|---|---|
|`-eq`|equal，等于|`[ $a -eq $b ]`|
|`-ne`|not equal，不等于|`[ $a -ne $b ]`|
|`-gt`|greater than，大于|`[ $a -gt $b ]`|
|`-lt`|less than，小于|`[ $a -lt $b ]`|
|`-ge`|greater or equal，大于等于|`[ $a -ge $b ]`|
|`-le`|less or equal，小于等于|`[ $a -le $b ]`|

#### 2.4.3 字符串运算符

用于字符串的判断与比较，配合`[ ]`/`[[ ]]`使用

|运算符|含义|示例|
|---|---|---|
|`=` / `==`|字符串相等，== 推荐在 [[]] 中使用|`[ "$str1" = "$str2" ]`|
|`!=`|字符串不相等|`[ "$str1" != "$str2" ]`|
|`-z`|zero，字符串长度为 0（空）|`[ -z "$str" ]`|
|`-n`|not zero，字符串长度非 0|`[ -n "$str" ]`|
|`>`|字符串大于（字典序）|`[[ "$str1" > "$str2" ]]`|
|`<`|字符串小于（字典序）|`[[ "$str1" < "$str2" ]]`|

#### 2.4.4 布尔逻辑运算符

| 运算符    | 含义         | 适用场景                  |
| ------ | ---------- | --------------------- |
| `!`    | 非，取反       | `[ ]`/`[[ ]]`通用       |
| `-a`   | 与，两边都为真才为真 | 仅`[ ]`中使用             |
| `-o`   | 或，一边为真就为真  | 仅`[ ]`中使用             |
| `&&`   | 与，两边都为真才为真 | 仅`[[ ]]`中使用，或命令之间短路执行 |
| `\|\|` | 或，一边为真就为真  | 仅`[[ ]]`中使用，或命令之间短路执行 |

#### 2.4.5 文件测试运算符（高频使用）

用于判断文件 / 目录的类型、权限、是否存在等，是 Shell 脚本中最常用的判断

表格

|运算符|核心含义|
|---|---|
|`-e`|文件 / 目录是否存在（exist）|
|`-f`|是否为普通文件（不是目录 / 设备文件）|
|`-d`|是否为目录（directory）|
|`-s`|文件是否存在且大小大于 0（非空）|
|`-r`|文件 / 目录是否有读权限|
|`-w`|文件 / 目录是否有写权限|
|`-x`|文件 / 目录是否有执行权限|
|`-L`|是否为软链接文件|
|`-f1 -nt f2`|文件 1 比文件 2 新（newer than）|
|`-f1 -ot f2`|文件 1 比文件 2 旧（older than）|

```bash
#!/bin/bash
# 文件测试示例
file="./test.txt"
dir="./test_dir"

# 判断文件是否存在
if [ -e "$file" ]; then
    echo "文件存在"
else
    echo "文件不存在，创建文件"
    touch "$file"
fi

# 判断是否为目录
if [ -d "$dir" ]; then
    echo "目录存在"
else
    echo "目录不存在，创建目录"
    mkdir -p "$dir"
fi
```

---

## 三、流程控制

### 3.1 条件判断

#### 3.1.1 核心语法：if 条件判断

##### 1. 单分支 if

```bash
if [ 条件表达式 ]; then
    # 条件成立执行的命令
fi
```

##### 2. 双分支 if-else

```bash
if [ 条件表达式 ]; then
    # 条件成立执行
else
    # 条件不成立执行
fi
```

##### 3. 多分支 if-elif-else

```bash
if [ 条件表达式1 ]; then
    # 条件1成立执行
elif [ 条件表达式2 ]; then
    # 条件2成立执行
else
    # 所有条件都不成立执行
fi
```

##### 4. [] 与 [[]] 的核心区别（重点避坑）

| 特性         | `[ ]`（test 命令）      | `[[ ]]`（Bash 增强）   |
| ---------- | ------------------- | ------------------ |
| 兼容性        | POSIX 标准，全 Shell 兼容 | Bash/Zsh 专属，sh 不支持 |
| 逻辑运算符      | 必须用`-a`/`-o`        | 支持`&&`/``，更直观      |
| 字符串比较`> <` | 需要转义`\>`/`\<`       | 无需转义，直接使用          |
| 正则匹配       | 不支持                 | 支持`=~`正则匹配         |
| 变量空格容错     | 变量必须加双引号，否则空格会报错    | 无需双引号，容错性更高        |
| 通配符匹配      | 不支持                 | 支持`*`/`?`通配符匹配     |

```bash
#!/bin/bash
# [[ ]] 高级用法示例
age=25
name="张三"

# 多条件判断
if [[ $age -gt 18 && $age -lt 30 ]]; then
    echo "年龄符合要求"
fi

# 通配符匹配
if [[ $name == 张* ]]; then
    echo "姓张"
fi

# 正则匹配（判断是否为数字）
num="123456"
if [[ $num =~ ^[0-9]+$ ]]; then
    echo "是纯数字"
fi
```

#### 3.1.2 case 分支判断

适用于多值匹配场景，比 if-elif 更简洁，类似其他语言的 switch-case

```bash
case 变量/表达式 in
    匹配模式1)
        # 匹配成功执行的命令
        ;;  # 双分号表示分支结束
    匹配模式2)
        # 匹配成功执行的命令
        ;;
    *)  # 通配符，匹配所有其他情况，类似default
        # 无匹配时执行
        ;;
esac
```

```bash
#!/bin/bash
# case示例：根据输入的数字输出星期
read -p "请输入1-7的数字：" week_num

case $week_num in
    1)
        echo "星期一"
        ;;
    2)
        echo "星期二"
        ;;
    3)
        echo "星期三"
        ;;
    4)
        echo "星期四"
        ;;
    5)
        echo "星期五"
        ;;
    6)
        echo "星期六"
        ;;
    7)
        echo "星期日"
        ;;
    *)
        echo "输入错误，请输入1-7的数字"
        ;;
esac
```

### 3.2 循环语句

#### 3.2.1 for 循环

适用于已知循环次数 / 遍历列表的场景，两种常用写法

##### 写法 1：for-in 遍历（最常用）

```bash
for 变量 in 取值列表
do
    # 循环体
done
```

```bash
#!/bin/bash
# 示例1：遍历数字序列
echo "===== 1-5循环 ====="
for i in {1..5}
do
    echo "数字：$i"
done

# 示例2：遍历文件
echo "===== 遍历当前目录.sh文件 ====="
for file in *.sh
do
    echo "脚本文件：$file"
done

# 示例3：遍历数组
arr=("Java" "Python" "Shell" "Go")
echo "===== 遍历数组 ====="
for lang in "${arr[@]}"
do
    echo "编程语言：$lang"
done
```

##### 写法 2：C 语言风格 for 循环

```bash
for (( 初始值; 循环条件; 变量更新 ))
do
    # 循环体
done
```

```bash
#!/bin/bash
# 示例：1-10求和
sum=0
for ((i=1; i<=10; i++))
do
    ((sum += i))
done
echo "1-10的和：$sum"
```

#### 3.2.2 while 循环

适用于未知循环次数，满足条件就循环的场景，条件不成立时退出

```bash
while [ 条件表达式 ]
do
    # 循环体
done
```

```bash
#!/bin/bash
# 示例1：1-10求和
sum=0
i=1
while [ $i -le 10 ]
do
    ((sum += i))
    ((i++))
done
echo "1-10的和：$sum"

# 示例2：逐行读取文件内容（高频使用）
file="./test.txt"
while read -r line
do
    echo "文件内容行：$line"
done < "$file"
```

#### 3.2.3 until 循环

与 while 相反，**条件不成立时循环，条件成立时退出**，适用于等待某个条件满足的场景

```bash
until [ 条件表达式 ]
do
    # 循环体
done
```

```bash
#!/bin/bash
# 示例：等待服务启动
count=0
until ping -c 1 baidu.com &> /dev/null
do
    echo "网络不通，等待重试..."
    sleep 1
    ((count++))
    if [ $count -ge 5 ]; then
        echo "重试5次失败，退出"
        exit 1
    fi
done
echo "网络通了！"
```

### 3.3 循环控制

表格

| 命令           | 作用                               |
| ------------ | -------------------------------- |
| `break`      | 终止整个循环，跳出循环体                     |
| `continue`   | 跳过当前循环的剩余代码，进入下一次循环              |
| `exit [状态码]` | 终止整个脚本，返回指定的状态码（0 = 成功，非 0 = 失败） |

bash

运行

```
#!/bin/bash
# 循环控制示例
for i in {1..10}
do
    # 跳过偶数
    if [ $((i % 2)) -eq 0 ]; then
        continue
    fi
    # 大于7时终止循环
    if [ $i -gt 7 ]; then
        break
    fi
    echo "当前数字：$i"
done

echo "循环结束"
# 退出脚本，返回状态码0
exit 0
```

---

## 四、函数与参数处理

Shell 函数是一段可重复调用的代码块，用于封装重复逻辑，提升脚本的可读性和可维护性。

### 4.1 函数的定义与调用

#### 语法格式

bash

运行

```
# 写法1：标准写法（推荐，兼容性最好）
function 函数名() {
    # 函数体
    # 执行命令
    # return 状态码（可选）
}

# 写法2：省略function关键字
函数名() {
    # 函数体
}

# 写法3：省略括号
function 函数名 {
    # 函数体
}
```

#### 调用规则

1. 函数必须**先定义，后调用**
2. 调用时直接写函数名，无需加括号
3. 函数参数直接跟在函数名后，用空格分隔

bash

运行

```
#!/bin/bash
# 函数定义
function say_hello() {
    echo "Hello Shell Function!"
}

# 函数调用
say_hello
```

### 4.2 函数参数与返回值

#### 4.2.1 函数参数

函数的参数使用和脚本参数一致，通过`$1、$2、$#、$@`等特殊变量获取

bash

运行

```
#!/bin/bash
# 带参数的函数
function get_sum() {
    # 获取两个参数
    num1=$1
    num2=$2
    # 计算和
    sum=$((num1 + num2))
    echo "两个数的和：$sum"
    echo "参数总个数：$#"
    echo "所有参数：$@"
}

# 调用函数，传入参数
get_sum 10 20
get_sum 30 40
```

#### 4.2.2 函数返回值

Shell 函数有两种返回方式，有严格的使用限制：

1. **return 语句**：只能返回**0-255 之间的整数**，用于表示函数的执行状态码，通过`$?`获取
2. **echo 语句**：可返回任意类型数据（字符串、大数、小数），通过`$(函数名)`捕获输出，是通用返回方式

bash

运行

```
#!/bin/bash
# 1. return返回状态码
function check_num() {
    num=$1
    if [ $num -gt 0 ]; then
        # 正数，返回0（成功）
        return 0
    else
        # 非正数，返回1（失败）
        return 1
    fi
}

# 调用函数
check_num 10
echo "正数判断结果：$?"  # 输出0

check_num -5
echo "负数判断结果：$?"  # 输出1

# 2. echo返回通用数据（推荐）
function get_sum2() {
    num1=$1
    num2=$2
    sum=$((num1 + num2))
    # 用echo输出结果，作为返回值
    echo $sum
}

# 捕获函数返回值
result=$(get_sum2 100 200)
echo "计算结果：$result"
echo "结果+50：$((result + 50))"
```

### 4.3 函数变量作用域

1. **全局变量**：脚本中默认定义的变量都是全局变量，整个脚本（包括函数内）都可访问和修改
2. **局部变量**：函数内用`local`关键字声明的变量，仅在函数内部生效，不会污染全局环境，**函数内定义变量推荐全部用 local**

bash

运行

```
#!/bin/bash
# 全局变量
name="全局张三"

function test_var() {
    # 局部变量，仅函数内生效
    local name="局部李四"
    local age=20
    echo "函数内name：$name"
    echo "函数内age：$age"
}

# 调用函数
test_var

# 全局变量不受影响
echo "函数外name：$name"
echo "函数外age：$age"  # 输出为空，局部变量外部无法访问
```

---

## 五、输入输出与重定向

### 5.1 标准输入输出与文件描述符

Linux 中一切皆文件，Shell 的输入输出都通过文件描述符管理，默认打开 3 个标准文件描述符：

表格

|文件描述符|名称|类型|默认设备|
|---|---|---|---|
|0|stdin|标准输入|键盘|
|1|stdout|标准输出|终端屏幕|
|2|stderr|标准错误输出|终端屏幕|

### 5.2 重定向核心用法

重定向：改变输入 / 输出的默认方向，将输出写入文件，或从文件读取输入，而非终端。

表格

|重定向符号|作用|示例|
|---|---|---|
|`>`|标准输出覆盖重定向，文件不存在则创建，存在则清空原有内容|`echo "内容" > test.txt`|
|`>>`|标准输出追加重定向，文件不存在则创建，存在则在末尾追加|`echo "内容" >> test.txt`|
|`2>`|标准错误覆盖重定向，仅重定向错误信息|`ls nofile 2> error.log`|
|`2>>`|标准错误追加重定向|`ls nofile 2>> error.log`|
|`&>` / `>&`|标准输出 + 标准错误 合并覆盖重定向|`ls test.txt nofile &> all.log`|
|`&>>`|标准输出 + 标准错误 合并追加重定向|`ls test.txt nofile &>> all.log`|
|`2>&1`|将标准错误重定向到标准输出的位置|`ls test.txt nofile > all.log 2>&1`|
|`<`|标准输入重定向，从文件读取输入，而非键盘|`wc -l < test.txt`|
|`/dev/null`|黑洞文件，写入的内容都会被丢弃，用于屏蔽输出|`ls test.txt &> /dev/null`|

bash

运行

```
#!/bin/bash
# 重定向示例
log_file="./run.log"

# 1. 输出写入文件
echo "脚本开始执行：$(date)" > "$log_file"

# 2. 追加写入日志
echo "执行第一步操作" >> "$log_file"
echo "执行第二步操作" >> "$log_file"

# 3. 屏蔽命令的所有输出
echo "执行静默操作" >> "$log_file"
ping -c 1 baidu.com &> /dev/null
if [ $? -eq 0 ]; then
    echo "网络正常" >> "$log_file"
else
    echo "网络异常" >> "$log_file"
fi

# 4. 错误和输出分开重定向
echo "执行文件操作" >> "$log_file"
ls test.txt 1>> "$log_file" 2>> error.log

echo "脚本执行结束：$(date)" >> "$log_file"
```

#### 5.2.1 Here Document 多行输入重定向

用于向命令 / 脚本传入多行文本，无需手动换行，语法：

bash

运行

```
命令 << 标记符
多行文本内容
标记符
```

- 标记符可自定义（常用 EOF/END），前后必须一致
- 结尾标记符必须顶格写，前后不能有任何字符
- 双引号包裹标记符，可禁用变量解析，原样输出内容

bash

运行

```
#!/bin/bash
# Here Document示例
# 1. 多行内容写入文件
cat > test.txt << EOF
===== 配置文件 =====
用户名：张三
年龄：20
日期：$(date)
====================
EOF

# 查看文件
cat test.txt

# 2. 原样输出，禁用变量解析
cat << "EOF"
这是原样输出，$name 不会被解析
EOF
```

### 5.3 管道符与 xargs

#### 5.3.1 管道符 |

将前一个命令的**标准输出**，作为后一个命令的**标准输入**，是 Shell 中串联命令的核心方式

bash

运行

```
# 语法
命令1 | 命令2 | 命令3 ...
```

bash

运行

```
# 示例1：统计当前目录.sh文件的个数
ls *.sh | wc -l

# 示例2：过滤包含error的日志行，统计行数
cat run.log | grep "error" | wc -l

# 示例3：排序并去重
cat test.txt | sort | uniq
```

#### 5.3.2 xargs 命令

管道符只能将前一个命令的输出作为后一个命令的标准输入，但很多命令（rm、cp、mkdir、kill 等）不支持标准输入，只支持命令行参数，`xargs`的核心作用就是**将标准输入转换成命令行参数**。

bash

运行

```
# 核心语法
命令1 | xargs [选项] 命令2
```

##### 常用选项

表格

|选项|作用|
|---|---|
|`-n 数字`|每次传递的参数个数|
|`-I 占位符`|用占位符代替传递的参数，可控制参数位置|
|`-d 分隔符`|指定输入的分隔符，默认是空格 / 换行|

bash

运行

```
#!/bin/bash
# xargs示例
# 1. 批量删除.log文件
find . -name "*.log" | xargs rm -f

# 2. 批量创建目录
echo "dir1 dir2 dir3" | xargs mkdir -p

# 3. -I 控制参数位置，批量重命名
ls *.txt | xargs -I {} mv {} {}.bak

# 4. -n 控制每次传递的参数个数
echo "1 2 3 4 5" | xargs -n 2 echo

# 5. 批量杀死进程
ps -ef | grep "nginx" | grep -v grep | awk '{print $2}' | xargs kill -9
```

---

## 六、Shell 三剑客（grep/sed/awk）

grep、sed、awk 是 Linux 文本处理的三大核心工具，是 Shell 脚本处理文本、日志、配置文件的灵魂，掌握这三个工具，能解决 99% 的文本处理需求。

### 6.1 grep 文本搜索

**核心作用**：在文件 / 输入流中，按正则表达式匹配行，过滤出符合条件的内容。

bash

运行

```
# 基础语法
grep [选项] "匹配模式" 文件名/输入流
```

#### 高频常用选项

表格

|选项|核心作用|
|---|---|
|`-v`|反向匹配，输出不包含匹配内容的行|
|`-i`|忽略大小写匹配|
|`-n`|输出匹配行的行号|
|`-c`|仅输出匹配到的总行数|
|`-w`|精确匹配整个单词，而非部分字符|
|`-E`|支持扩展正则表达式（等同于 egrep）|
|`-F`|按固定字符串匹配，不解析正则（等同于 fgrep）|
|`-o`|仅输出匹配到的内容，而非整行|
|`-A n`|输出匹配行及后面 n 行|
|`-B n`|输出匹配行及前面 n 行|
|`-C n`|输出匹配行及前后各 n 行|

bash

运行

```
# 示例1：在test.txt中搜索包含"error"的行，显示行号
grep -n "error" test.txt

# 示例2：反向匹配，输出不包含"#"注释的行
grep -v "^#" nginx.conf

# 示例3：忽略大小写，匹配包含"hello"的行
grep -i "hello" test.txt

# 示例4：精确匹配单词"root"，避免匹配到"root123"
grep -w "root" /etc/passwd

# 示例5：统计包含"error"的行数
grep -c "error" run.log

# 示例6：扩展正则，匹配以a开头或以b结尾的行
grep -E "^a|b$" test.txt
```

### 6.2 sed 文本流编辑

**核心作用**：流式文本编辑器，按行处理文件 / 输入流，实现文本的增、删、改、查，无需打开文件，适合脚本批量处理文件。

bash

运行

```
# 基础语法
sed [选项] '操作指令' 文件名/输入流
```

#### 核心说明

- 默认仅输出处理后的内容，**不会修改原文件**，加`-i`选项可直接修改原文件（生产环境慎用，建议先备份）
- 操作指令格式：`[地址范围]操作类型[参数]`

#### 高频常用选项

表格

|选项|核心作用|
|---|---|
|`-i`|直接修改原文件（in-place）|
|`-n`|静默模式，仅输出被处理的行（配合 p 指令使用）|
|`-e`|执行多个操作指令|
|`-r`|支持扩展正则表达式|

#### 高频操作指令

表格

|指令|作用|示例|
|---|---|---|
|`s`|替换（最常用）|`s/旧内容/新内容/`|
|`p`|打印行|`2p` 打印第 2 行|
|`d`|删除行|`3d` 删除第 3 行|
|`a`|行后追加内容|`2a 新内容` 第 2 行后追加|
|`i`|行前插入内容|`2i 新内容` 第 2 行前插入|

bash

运行

```
# 1. 文本替换（核心高频）
# 替换每行第一个匹配的root为admin
sed 's/root/admin/' test.txt
# 全局替换，替换所有匹配的root为admin
sed 's/root/admin/g' test.txt
# 替换第3行的root为admin
sed '3s/root/admin/g' test.txt
# 替换1-5行的root为admin
sed '1,5s/root/admin/g' test.txt
# 忽略大小写替换
sed 's/root/admin/gi' test.txt
# 直接修改原文件（生产环境先备份！）
sed -i 's/root/admin/g' test.txt
# 备份原文件为.bak，再修改原文件
sed -i.bak 's/root/admin/g' test.txt

# 2. 行删除
# 删除第2行
sed '2d' test.txt
# 删除空行
sed '/^$/d' test.txt
# 删除1-3行
sed '1,3d' test.txt
# 删除包含"error"的行
sed '/error/d' test.txt
# 删除最后一行
sed '$d' test.txt

# 3. 行打印（配合-n）
# 打印第2行
sed -n '2p' test.txt
# 打印1-5行
sed -n '1,5p' test.txt
# 打印包含"hello"的行（等同于grep）
sed -n '/hello/p' test.txt

# 4. 行追加与插入
# 第2行后追加新内容
sed '2a 这是追加的内容' test.txt
# 第1行前插入内容
sed '1i 这是插入的标题' test.txt
# 最后一行追加内容
sed '$a 这是结尾内容' test.txt

# 5. 多个操作（-e）
sed -e 's/root/admin/g' -e '/^#/d' test.txt
```

### 6.3 awk 文本处理与分析

**核心作用**：强大的文本处理编程语言，擅长按字段（列）处理文本，支持条件判断、循环、数组、统计运算，适合日志分析、数据统计、报表生成等复杂场景。

bash

运行

```
# 基础语法
awk [选项] 'BEGIN{初始化操作} 模式匹配{处理动作} END{收尾操作}' 文件名/输入流
```

#### 核心概念

1. **记录（Record）**：awk 默认将每行作为一条记录，分隔符是换行符
    
2. **字段（Field）**：awk 默认将每行按空格 / 制表符分割成多个字段，`$1`代表第 1 列，`$2`代表第 2 列，`$0`代表整行内容
    
3. **内置变量**：
    
    表格
    
    |内置变量|核心含义|
    |---|---|
    |`FS`|输入字段分隔符，默认空格 / 制表符|
    |`OFS`|输出字段分隔符，默认空格|
    |`RS`|输入记录分隔符，默认换行符|
    |`ORS`|输出记录分隔符，默认换行符|
    |`NF`|当前行的字段总数（Number of Fields），`$NF`代表最后一列|
    |`NR`|当前处理的行号（Number of Records）|
    |`FNR`|多文件处理时，每个文件单独的行号|
    |`FILENAME`|当前处理的文件名|
    
4. **执行流程**：
    
    1. 执行`BEGIN{}`块中的代码（仅执行一次，在读取文件前执行，常用于初始化变量、设置分隔符）
    2. 逐行读取文件，执行`模式匹配{处理动作}`，匹配成功则执行对应动作
    3. 所有行处理完毕，执行`END{}`块中的代码（仅执行一次，常用于输出统计结果）
    

bash

运行

```
# 基础示例
# 1. 打印/etc/passwd的第1列（用户名）和第7列（默认Shell）
awk -F ":" '{print $1, $7}' /etc/passwd
# -F ":" 指定分隔符为冒号

# 2. 打印第2行的内容
awk 'NR==2{print $0}' test.txt

# 3. 打印最后一列
awk '{print $NF}' test.txt

# 4. 打印行号和内容
awk '{print NR, $0}' test.txt

# 5. 条件过滤：打印第3列大于100的行
awk '$3 > 100{print $0}' test.txt

# 6. 统计文件行数（等同于wc -l）
awk 'END{print NR}' test.txt

# 7. 求和：统计第2列的总和
awk 'BEGIN{sum=0} {sum += $2} END{print "总和：", sum}' test.txt

# 8. 求平均值
awk 'BEGIN{sum=0} {sum += $2} END{print "平均值：", sum/NR}' test.txt

# 9. 多条件匹配
awk -F ":" '$3>=1000 && $7=="/bin/bash"{print $1}' /etc/passwd
# 筛选出普通用户（UID>=1000）且默认Shell是bash的用户名

# 10. 数组统计：统计日志中IP出现的次数（高频日志分析）
awk '{ip[$1]++} END{for(i in ip) print i, ip[i]}' access.log
# 访问日志第1列是IP，统计每个IP的访问次数
```

---

## 七、脚本进阶与健壮性

### 7.1 脚本调试方法

#### 7.1.1 命令行调试

执行脚本时添加调试参数，无需修改脚本内容

bash

运行

```
# 1. 跟踪执行过程，打印每一行执行的命令和结果（最常用）
bash -x hello.sh

# 2. 检查脚本语法错误，不执行脚本（语法检查）
bash -n hello.sh

# 3. 详细模式，打印脚本读取的所有行
bash -v hello.sh
```

#### 7.1.2 脚本内调试（set 命令）

在脚本内通过`set`命令开启 / 关闭调试，精准控制调试范围，同时可增强脚本健壮性

bash

运行

```
#!/bin/bash
# 脚本开头推荐配置（生产环境必备）
# -e：脚本中任何命令执行失败（退出码非0），立即终止脚本
# -u：遇到未定义的变量，立即报错并终止脚本（避免变量为空导致的灾难）
# -o pipefail：管道中任意一个命令失败，整个管道的退出码为非0，脚本终止
# -x：开启调试模式，打印执行的每一行命令
set -euo pipefail
# 如需调试，打开下面注释
# set -x

# 脚本内容
echo "脚本开始"

# 临时关闭调试
set +x
echo "这部分不会被调试"
# 重新开启调试
set -x

echo "脚本结束"
```

### 7.2 信号处理与 trap

`trap`命令用于捕获 Shell 信号，在脚本接收到指定信号时，执行预设的操作，常用于脚本退出时清理临时文件、处理用户中断（Ctrl+C）、记录日志等，提升脚本的健壮性。

bash

运行

```
# 基础语法
trap "执行的命令" 信号1 信号2 ...
```

#### 常用信号

表格

|信号|编号|含义|
|---|---|---|
|`SIGINT`|2|键盘中断（Ctrl+C）|
|`SIGTERM`|15|程序终止信号（默认 kill 命令发送）|
|`SIGEXIT`|0|脚本正常退出时触发|
|`SIGERR`|-|命令执行失败时触发|

bash

运行

```
#!/bin/bash
set -euo pipefail

# 定义临时文件
temp_file="./temp.tmp"
log_file="./script.log"

# 捕获信号：脚本退出时，删除临时文件，记录退出日志
trap 'echo "脚本退出，清理临时文件"; rm -f $temp_file; echo "脚本执行结束：$(date)" >> $log_file' EXIT

# 捕获Ctrl+C中断信号
trap 'echo "用户按下Ctrl+C，终止脚本"; exit 1' INT

# 脚本开始
echo "脚本开始执行：$(date)" > $log_file
echo "创建临时文件"
touch $temp_file

# 模拟业务逻辑
echo "执行业务操作..."
sleep 10
echo "业务操作完成"

exit 0
```

### 7.3 常用内置命令

#### 7.3.1 read 读取用户输入

用于从标准输入读取用户输入，赋值给变量，实现脚本与用户的交互

bash

运行

```
# 基础语法
read [选项] 变量名
```

表格

|选项|核心作用|
|---|---|
|`-p "提示信息"`|输出提示信息，等待用户输入|
|`-s`|静默模式，不显示用户输入的内容（适合输入密码）|
|`-t 秒数`|超时时间，超时自动退出|
|`-n 数字`|限制输入的字符个数，达到个数自动结束|

bash

运行

```
#!/bin/bash
# 读取用户名
read -p "请输入您的用户名：" username
echo "您输入的用户名：$username"

# 读取密码，静默模式
read -s -p "请输入您的密码：" password
echo -e "\n您输入的密码长度：${#password}"

# 超时10秒
read -t 10 -p "请在10秒内输入内容：" content
echo -e "\n您输入的内容：$content"
```

#### 7.3.2 shift 位置参数偏移

用于将脚本 / 函数的位置参数向左偏移，默认偏移 1 位，常用于处理多个不定数量的参数

bash

运行

```
#!/bin/bash
echo "所有参数：$@"
echo "第一个参数：$1"

# 偏移1位
shift
echo "偏移后所有参数：$@"
echo "偏移后第一个参数：$1"

# 偏移2位
shift 2
echo "再偏移2位后所有参数：$@"
```

#### 7.3.3 其他常用内置命令

表格

|命令|核心作用|
|---|---|
|`echo`|输出内容到控制台，`-e`支持转义字符|
|`printf`|格式化输出，类似 C 语言 printf，精准控制输出格式|
|`unset`|删除变量 / 数组|
|`export`|声明环境变量|
|`readonly`|声明只读变量|
|`ulimit`|控制系统资源限制|
|`umask`|设置文件 / 目录的默认权限掩码|

### 7.4 正则表达式基础

Shell 三剑客、条件判断都依赖正则表达式，核心分为基础正则（BRE）和扩展正则（ERE）

#### 基础正则元字符

表格

|元字符|含义|
|---|---|
|`^`|匹配行的开头|
|`$`|匹配行的结尾|
|`.`|匹配任意单个字符|
|`*`|匹配前一个字符 0 次或多次|
|`[]`|匹配括号内的任意单个字符，`[0-9]`匹配数字，`[a-z]`匹配小写字母|
|`[^]`|反向匹配，匹配不在括号内的任意字符|
|`\`|转义字符，取消元字符的特殊含义|
|`\{n\}`|匹配前一个字符 n 次|
|`\{n,\}`|匹配前一个字符至少 n 次|
|`\{n,m\}`|匹配前一个字符 n 到 m 次|

#### 扩展正则元字符（grep -E/sed -r/awk 支持）

表格

|元字符|含义||
|---|---|---|
|`+`|匹配前一个字符 1 次或多次||
|`?`|匹配前一个字符 0 次或 1 次||
|`|`|或，匹配左右任意一个表达式|
|`()`|分组，将多个字符作为一个整体||
|`{n}`|匹配前一个字符 n 次（无需转义）||

bash

运行

```
# 正则示例
# 1. 匹配空行
grep "^$" test.txt

# 2. 匹配以#开头的注释行
grep "^#" test.txt

# 3. 匹配以数字结尾的行
grep "[0-9]$" test.txt

# 4. 匹配手机号（11位数字，1开头）
grep -E "^1[3-9][0-9]{9}$" phone.txt

# 5. 匹配邮箱地址
grep -E "^[a-zA-Z0-9_-]+@[a-zA-Z0-9_-]+(\.[a-zA-Z0-9_-]+)+$" email.txt
```

---

## 八、企业级实战脚本案例

### 案例 1：系统资源监控告警脚本

监控 CPU、内存、磁盘、系统负载，超过阈值输出告警，可对接邮件 / 企业微信告警

bash

运行

```
#!/bin/bash
set -euo pipefail

# 告警阈值配置
CPU_WARN=80          # CPU使用率告警阈值%
MEM_WARN=90          # 内存使用率告警阈值%
DISK_WARN=90         # 磁盘使用率告警阈值%
LOAD_WARN=$(grep -c 'processor' /proc/cpuinfo)  # 系统负载阈值=CPU核心数

# 日志文件
LOG_FILE="./system_monitor.log"

# 日志函数
function log() {
    echo "[$(date "+%Y-%m-%d %H:%M:%S")] $1" >> "$LOG_FILE"
    echo "[$(date "+%Y-%m-%d %H:%M:%S")] $1"
}

# 1. CPU使用率监控
function check_cpu() {
    # 取1分钟平均CPU使用率，取空闲率计算使用率
    cpu_idle=$(top -bn1 | grep Cpu | awk '{print $8}' | cut -d. -f1)
    cpu_usage=$((100 - cpu_idle))
    log "当前CPU使用率：${cpu_usage}%"

    if [ $cpu_usage -ge $CPU_WARN ]; then
        log "【告警】CPU使用率超过阈值${CPU_WARN}%！"
    fi
}

# 2. 内存使用率监控
function check_mem() {
    mem_total=$(free -m | grep Mem | awk '{print $2}')
    mem_used=$(free -m | grep Mem | awk '{print $3}')
    mem_usage=$((mem_used * 100 / mem_total))
    log "当前内存使用率：${mem_usage}%，总内存：${mem_total}MB，已使用：${mem_used}MB"

    if [ $mem_usage -ge $MEM_WARN ]; then
        log "【告警】内存使用率超过阈值${MEM_WARN}%！"
    fi
}

# 3. 磁盘使用率监控
function check_disk() {
    log "===== 磁盘使用率 ====="
    df -h | grep -vE 'tmpfs|overlay|loop' | awk '{print $5 " " $6}' | while read usage mountpoint
    do
        usage_num=$(echo $usage | cut -d% -f1)
        log "挂载点${mountpoint}使用率：${usage}"
        if [ $usage_num -ge $DISK_WARN ]; then
            log "【告警】挂载点${mountpoint}使用率超过阈值${DISK_WARN}%！"
        fi
    done
}

# 4. 系统负载监控
function check_load() {
    load_1min=$(uptime | awk -F 'load average:' '{print $2}' | cut -d, -f1 | sed 's/ //g')
    log "当前系统1分钟平均负载：${load_1min}，阈值：${LOAD_WARN}"

    # 浮点比较用bc
    if [ $(echo "$load_1min > $LOAD_WARN" | bc) -eq 1 ]; then
        log "【告警】系统负载超过阈值${LOAD_WARN}！"
    fi
}

# 主函数
function main() {
    log "===== 开始系统资源监控 ====="
    check_cpu
    check_mem
    check_disk
    check_load
    log "===== 系统资源监控结束 ====="
}

# 执行主函数
main
```

### 案例 2：服务健康检查与自动重启脚本

检查指定服务是否运行，异常时自动重启并记录日志

bash

运行

```
#!/bin/bash
set -euo pipefail

# 配置要监控的服务
SERVICES=("nginx" "mysqld" "sshd")
# 重启失败重试次数
RETRY_MAX=3
# 日志文件
LOG_FILE="./service_check.log"

# 日志函数
function log() {
    echo "[$(date "+%Y-%m-%d %H:%M:%S")] $1" >> "$LOG_FILE"
}

# 检查服务状态
function check_service() {
    service_name=$1
    # 检查服务是否运行
    if systemctl is-active --quiet "$service_name"; then
        log "服务${service_name}运行正常"
        return 0
    else
        log "【异常】服务${service_name}未运行，准备重启"
        return 1
    fi
}

# 重启服务
function restart_service() {
    service_name=$1
    retry_count=0

    while [ $retry_count -lt $RETRY_MAX ]
    do
        log "尝试重启服务${service_name}，第$((retry_count+1))次"
        systemctl restart "$service_name"
        sleep 2

        # 检查重启是否成功
        if systemctl is-active --quiet "$service_name"; then
            log "服务${service_name}重启成功"
            return 0
        fi

        ((retry_count++))
    done

    log "【严重告警】服务${service_name}重启${RETRY_MAX}次全部失败！"
    return 1
}

# 主函数
function main() {
    log "===== 开始服务健康检查 ====="
    for service in "${SERVICES[@]}"
    do
        if ! check_service "$service"; then
            restart_service "$service"
        fi
    done
    log "===== 服务健康检查结束 ====="
}

# 执行主函数
main
```

### 案例 3：日志自动切割与清理脚本

定期切割大日志文件，清理过期日志，避免磁盘占满

bash

运行

```
#!/bin/bash
set -euo pipefail

# 配置
LOG_DIR="/var/log/nginx"  # 日志目录
ARCHIVE_DIR="$LOG_DIR/archive"  # 归档目录
KEEP_DAYS=7  # 保留7天的归档日志
LOG_FILES=("access.log" "error.log")  # 要切割的日志文件

# 创建归档目录
mkdir -p "$ARCHIVE_DIR"

# 日志函数
function log() {
    echo "[$(date "+%Y-%m-%d %H:%M:%S")] $1"
}

# 切割日志
function cut_log() {
    log_file=$1
    # 归档文件名：日志名_年月日时分.log
    archive_name="${log_file%.*}_$(date "+%Y%m%d%H%M").log"
    archive_path="$ARCHIVE_DIR/$archive_name"

    # 检查日志文件是否存在
    if [ ! -f "$LOG_DIR/$log_file" ]; then
        log "日志文件$LOG_DIR/$log_file不存在，跳过"
        return 1
    fi

    # 复制日志到归档目录
    cp "$LOG_DIR/$log_file" "$archive_path"
    # 清空原日志文件（避免服务句柄不释放）
    > "$LOG_DIR/$log_file"
    log "日志$log_file切割完成，归档到$archive_path"

    # 压缩归档文件
    gzip "$archive_path"
    log "归档文件压缩完成：${archive_path}.gz"
}

# 清理过期日志
function clean_expired_log() {
    log "开始清理${KEEP_DAYS}天前的归档日志"
    find "$ARCHIVE_DIR" -name "*.log.gz" -mtime +$KEEP_DAYS -delete
    if [ $? -eq 0 ]; then
        log "过期日志清理完成"
    else
        log "过期日志清理失败"
    fi
}

# 主函数
function main() {
    log "===== 开始日志切割 ====="
    for log_file in "${LOG_FILES[@]}"
    do
        cut_log "$log_file"
    done
    clean_expired_log
    log "===== 日志切割结束 ====="
}

# 执行主函数
main
```

---

## 九、Shell 脚本最佳实践与避坑指南

### 9.1 最佳实践

1. **脚本开头规范**
    
    - 必须写`#!/bin/bash`，不要写`#!/bin/sh`，避免兼容性问题
    - 推荐添加`set -euo pipefail`，提升脚本健壮性，提前暴露问题
    - 添加脚本说明、作者、创建时间、修改记录等注释
    
2. **变量规范**
    
    - 变量名见名知意，避免单字符变量（循环变量 i/j 除外）
    - 环境变量全大写，自定义变量小写，避免与系统变量冲突
    - 函数内的变量全部用`local`声明，避免污染全局环境
    - 所有变量引用都用双引号包裹`"$var"`，避免空格 / 特殊字符导致的语法错误
    - 变量引用推荐用`${var}`格式，避免字符串拼接歧义
    
3. **代码规范**
    
    - 统一缩进（4 个空格），提升代码可读性
    - 重复逻辑封装成函数，避免代码冗余
    - 关键逻辑添加注释，说明业务含义
    - 脚本执行过程输出日志，记录关键步骤和异常，方便排查问题
    - 路径尽量使用绝对路径，避免相对路径导致的执行异常
    - 命令执行后判断退出码`$?`，处理异常情况，避免脚本继续执行导致灾难
    
4. **安全规范**
    
    - 不要在脚本中硬编码密码、密钥等敏感信息
    - 校验用户输入的内容，避免注入攻击
    - 生产环境执行`rm -rf`等高危命令前，必须做路径校验，禁止`rm -rf $var/`（$var 为空时会变成`rm -rf /`）
    - 脚本设置最小权限，避免给 777 权限
    

### 9.2 高频避坑指南

1. **赋值等号两侧加空格**
    
    - 错误：`name = "张三"`
    - 正确：`name="张三"`
    
2. **变量引用不加双引号，导致空格异常**
    
    - 错误：`rm $file`（file 包含空格时会被拆分成多个文件）
    - 正确：`rm "$file"`
    
3. **[ ] 条件判断中变量不加双引号，空值导致语法错误**
    
    - 错误：`[ $name = "张三" ]`（name 为空时变成`[ = "张三" ]`，语法报错）
    - 正确：`[ "$name" = "张三" ]` 或 `[[ $name = "张三" ]]`
    
4. **函数 return 返回字符串 / 大数**
    
    - 错误：return 返回字符串，或大于 255 的数字
    - 正确：用 echo 输出返回值，通过`$()`捕获
    
5. **`$*`和`$@`混淆，遍历参数异常**
    
    - 遍历参数必须用`"$@"`，不要用`"$*"`，否则所有参数会被合并成一个字符串
    
6. **Shell 默认整数运算，浮点运算直接用`$(( ))`**
    
    - 错误：`$((10/3))` 只会得到 3
    - 正确：用 bc 工具`echo "scale=2; 10/3" | bc`
    
7. **管道符中修改变量，外部无法获取**
    
    - 管道符会开启子 Shell，子 Shell 内修改的变量不会影响父 Shell，避免在管道循环中修改全局变量
    
8. **sed -i 直接修改原文件，无备份**
    
    - 生产环境使用 sed -i 前，先备份原文件，可用`sed -i.bak`自动备份
    
9. **使用反引号 `` 代替 $()**
    
    - 反引号不支持嵌套，可读性差，推荐全部用`$()`执行命令替换
    
10. **脚本中使用相对路径，执行目录变化导致异常**
    
    - 推荐在脚本开头切换到脚本所在目录：`cd $(dirname $0)` 或使用绝对路径
    

---

## 十、附录：常用速查表

### 10.1 常用内置变量速查

表格

|变量|含义|
|---|---|
|`$0`|脚本文件名|
|`$1~$n`|第 1~n 个参数|
|`$#`|参数总个数|
|`$@`|所有参数，独立字符串|
|`$*`|所有参数，合并为一个字符串|
|`$?`|上一条命令退出码|
|`$$`|当前进程 PID|
|`$!`|后台最后一个进程 PID|
|`$HOME`|当前用户家目录|
|`$PATH`|命令搜索路径|
|`$PWD`|当前工作目录|
|`$USER`|当前用户名|
|`$SHELL`|当前默认 Shell|
|`$LANG`|系统语言编码|

### 10.2 常用命令速查

表格

|分类|常用命令|
|---|---|
|文件目录|ls、cd、pwd、mkdir、rm、cp、mv、touch、find、chmod、chown、ln|
|文本处理|cat、more、less、head、tail、grep、sed、awk、sort、uniq、wc、cut、tr|
|系统监控|top、htop、free、df、du、uptime、ps、netstat、ss、iostat、vmstat|
|网络相关|ping、telnet、curl、wget、ifconfig、ip、route、netstat、ss、tcpdump|
|压缩解压|tar、zip、unzip、gzip、bzip2|
|权限相关|chmod、chown、chgrp、umask、sudo、su|
|进程管理|ps、top、kill、killall、systemctl、service、jobs、fg、bg、nohup|