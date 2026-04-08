
Python 标准库是 Python 安装时自带的内置模块集合，无需额外安装，经过严格的测试与优化，覆盖系统交互、时间处理、数学计算、数据序列化、正则匹配、数据结构扩展等全场景，是 Python「自带电池」设计哲学的核心体现。本节覆盖目录指定的核心标准库，讲解高频用法、核心特性与最佳实践。

### 6.6.1 os 模块：操作系统接口

`os`模块是 Python 与操作系统交互的核心模块，提供了跨平台的操作系统功能调用，包括文件 / 目录操作、进程管理、环境变量、权限管理等核心功能。

高频核心功能分类：

1. **路径与目录操作**

    |方法|功能描述|
    |---|---|
    |`os.getcwd()`|获取当前工作目录的绝对路径|
    |`os.chdir(path)`|切换当前工作目录到指定路径|
    |`os.listdir(path)`|列出指定目录下的所有文件和子目录名，返回列表|
    |`os.mkdir(path)`|创建单级目录，目录已存在抛出`FileExistsError`|
    |`os.makedirs(path, exist_ok=False)`|递归创建多级目录，`exist_ok=True`时目录已存在不报错|
    |`os.rmdir(path)`|删除空目录|
    |`os.removedirs(path)`|递归删除空目录|
    |`os.remove(path)`|删除指定文件|
    |`os.rename(src, dst)`|重命名文件 / 目录|
    |`os.path.exists(path)`|判断路径是否存在，返回布尔值|
    |`os.path.isfile(path)`|判断路径是否为文件|
    |`os.path.isdir(path)`|判断路径是否为目录|
    |`os.path.abspath(path)`|返回路径的绝对路径|
    |`os.path.basename(path)`|返回路径的文件名部分|
    |`os.path.dirname(path)`|返回路径的目录部分|
    |`os.path.join(path1, path2, ...)`|跨平台拼接路径，自动适配系统路径分隔符|
    |`os.path.getsize(path)`|获取文件大小，单位为字节|
    
    补充：Python 3.4 + 推荐使用`pathlib`模块（面向对象的路径操作）替代`os.path`，语法更简洁、可读性更强。
    
2. **进程与环境变量操作**

    |方法|功能描述|
    |---|---|
    |`os.system(command)`|执行系统 shell 命令，返回命令退出状态码|
    |`os.environ`|系统环境变量的字典对象，可读取 / 修改环境变量|
    |`os.getenv(key, default=None)`|获取指定环境变量的值，不存在返回默认值|
    |`os.putenv(key, value)`|设置系统环境变量|
    |`os.getpid()`|获取当前进程的 PID|
    |`os.getppid()`|获取当前进程的父进程 PID|
    |`os.kill(pid, signal)`|向指定 PID 的进程发送信号|
    
3. **权限管理**

    |方法|功能描述|
    |---|---|
    |`os.chmod(path, mode)`|修改文件 / 目录的权限，如`0o755`|
    |`os.chown(path, uid, gid)`|修改文件 / 目录的所属用户与用户组|
    |`os.access(path, mode)`|测试当前用户对路径的访问权限|
    

### 6.6.2 sys 模块：Python 解释器交互

`sys`模块用于与 Python 解释器进行交互，提供了解释器的运行时环境、配置、参数、退出等核心控制能力。

高频核心功能：

|方法 / 属性|功能描述|
|---|---|
|`sys.path`|模块搜索路径列表，可手动添加自定义路径|
|`sys.argv`|命令行参数列表，`sys.argv[0]`为脚本文件名，后续元素为传入的参数|
|`sys.modules`|已加载模块的缓存字典，键为模块名，值为模块对象|
|`sys.version`|Python 解释器的版本信息字符串|
|`sys.version_info`|Python 版本的元组对象，可用于版本判断，如`sys.version_info >= (3, 8)`|
|`sys.platform`|操作系统平台标识，如`win32`、`linux`、`darwin`（macOS）|
|`sys.exit(code=0)`|退出 Python 解释器，code 为退出状态码，0 表示正常退出，非 0 表示异常退出|
|`sys.stdin` / `sys.stdout` / `sys.stderr`|标准输入、标准输出、标准错误流对象，可重定向输入输出|
|`sys.getrefcount(obj)`|获取对象的引用计数，用于垃圾回收调试|
|`sys.getsizeof(obj)`|获取对象占用的内存大小，单位为字节|

### 6.6.3 datetime 模块：日期时间处理

`datetime`模块是 Python 处理日期、时间、时间差的核心标准库，提供了面向对象的日期时间处理能力，替代了老旧的`time`模块的大部分功能，支持时区、时间格式化、时间计算等核心能力。

核心类与高频用法：

1. **核心类**
 
    |类名|功能描述|
    |---|---|
    |`datetime.datetime`|日期时间类，包含年、月、日、时、分、秒、微秒，最核心的类|
    |`datetime.date`|日期类，仅包含年、月、日|
    |`datetime.time`|时间类，仅包含时、分、秒、微秒|
    |`datetime.timedelta`|时间差类，用于两个日期时间的加减计算|
    |`datetime.timezone`|时区类，用于实现带时区的时间（aware datetime）|
    
2. **核心用法**
    
    - 获取当前时间

        ```python
        from datetime import datetime, date
        
        # 获取当前本地日期时间（naive datetime，无时区信息）
        now = datetime.now()
        # 获取当前本地日期
        today = date.today()
        # 获取当前UTC时间
        utc_now = datetime.utcnow()
        ```
        
    - 时间差计算（timedelta）

        ```python
        from datetime import datetime, timedelta
        
        now = datetime.now()
        # 1天后的时间
        tomorrow = now + timedelta(days=1)
        # 1小时前的时间
        one_hour_ago = now - timedelta(hours=1)
        # 两个时间的差值，返回timedelta对象
        delta = tomorrow - now
        print(delta.days, delta.seconds)
        ```
        
    - 时间格式化与解析

        ```python
        from datetime import datetime
        
        now = datetime.now()
        # datetime → 格式化字符串，strftime
        format_str = now.strftime("%Y-%m-%d %H:%M:%S")
        # 输出示例：2024-05-20 14:30:00
        
        # 格式化字符串 → datetime，strptime
        time_str = "2024-05-20 14:30:00"
        dt = datetime.strptime(time_str, "%Y-%m-%d %H:%M:%S")
        ```
        
        核心格式化符：`%Y`4 位年份、`%m`2 位月份、`%d`2 位日期、`%H`24 小时制小时、`%M`分钟、`%S`秒。
        
    - 时区处理
        
        Python 的 datetime 分为**naive datetime**（无时区信息，默认本地时区）和**aware datetime**（带时区信息，无歧义），工程化开发中推荐使用带时区的时间，避免跨时区业务的时间错误。

        ```python
        from datetime import datetime, timezone, timedelta
        
        # 定义东八区时区（北京时间）
        tz_cst = timezone(timedelta(hours=8), name="CST")
        # 获取当前东八区时间（aware datetime）
        now_cst = datetime.now(tz=tz_cst)
        # 转换为UTC时间
        now_utc = now_cst.astimezone(timezone.utc)
        ```
        
    

### 6.6.4 math 模块：数学函数

`math`模块提供了 C 标准库中的基础数学函数，支持浮点数的数学运算，包括三角函数、对数、指数、幂运算、取整、常数等，适用于基础数值计算。

高频核心内容：

1. **数学常数**

    |常数|描述|
    |---|---|
    |`math.pi`|圆周率 π，精确到双精度浮点数|
    |`math.e`|自然对数的底数 e|
    |`math.inf`|正无穷大浮点数|
    |`math.nan`|非数字（Not a Number）浮点数|
    
2. **高频函数**

    |函数|功能描述|
    |---|---|
    |`math.sqrt(x)`|计算 x 的平方根|
    |`math.pow(x, y)`|计算 x 的 y 次幂，等价于`x**y`|
    |`math.exp(x)`|计算 e 的 x 次幂|
    |`math.log(x, base=math.e)`|计算对数，默认自然对数，可指定底数|
    |`math.log10(x)`|计算以 10 为底的对数|
    |`math.sin(x)` / `math.cos(x)` / `math.tan(x)`|三角函数，x 为弧度值|
    |`math.degrees(x)`|弧度转换为角度|
    |`math.radians(x)`|角度转换为弧度|
    |`math.ceil(x)`|向上取整，返回大于等于 x 的最小整数|
    |`math.floor(x)`|向下取整，返回小于等于 x 的最大整数|
    |`math.fabs(x)`|返回 x 的绝对值|
    |`math.factorial(x)`|计算 x 的阶乘|
    |`math.gcd(a, b)`|计算 a 和 b 的最大公约数|
    

补充：复数的数学运算需使用`cmath`模块，而非`math`模块。

### 6.6.5 random 模块：伪随机数生成

`random`模块提供了伪随机数生成功能，支持随机整数、随机浮点数、随机序列选择、序列打乱等操作，适用于普通的随机场景、模拟、测试等。

**核心注意**：`random`模块生成的是伪随机数，基于梅森旋转算法，不可用于密码学、加密等安全场景，安全随机数需使用`secrets`模块。

高频核心函数：

|函数|功能描述|
|---|---|
|`random.random()`|生成 [0.0, 1.0) 范围内的随机浮点数|
|`random.uniform(a, b)`|生成 [a, b] 范围内的随机浮点数|
|`random.randint(a, b)`|生成 [a, b] 范围内的随机整数，包含 a 和 b|
|`random.randrange(start, stop, step)`|生成 range (start, stop, step) 范围内的随机整数，不包含 stop|
|`random.choice(seq)`|从非空序列中随机选择一个元素|
|`random.choices(seq, weights=None, k=1)`|从序列中随机选择 k 个元素，可指定权重，支持重复选择|
|`random.sample(seq, k)`|从序列中随机选择 k 个不重复的元素，无放回抽样|
|`random.shuffle(seq)`|原地打乱序列的元素顺序，仅支持可变序列|
|`random.seed(a)`|设置随机数种子，固定种子后可生成固定的随机序列，用于复现结果|

### 6.6.6 json 模块：JSON 数据序列化与反序列化

`json`模块是 Python 处理 JSON 格式数据的核心标准库，实现了 Python 对象与 JSON 字符串的双向转换，是接口开发、数据传输、配置文件处理的核心工具。

JSON 与 Python 数据类型的映射关系：

|JSON 类型|Python 类型|
|---|---|
|object|dict|
|array|list / tuple|
|string|str|
|number (整数)|int|
|number (浮点数)|float|
|true / false|True / False|
|null|None|

核心函数：

1. **序列化：Python 对象 → JSON 字符串 / 文件**

    |函数|功能描述|
    |---|---|
    |`json.dumps(obj, ensure_ascii=True, indent=None, sort_keys=False, default=None)`|将 Python 对象序列化为 JSON 字符串|
    |`json.dump(obj, fp, ensure_ascii=True, indent=None, sort_keys=False, default=None)`|将 Python 对象序列化为 JSON 字符串，写入文件对象 fp|
    
    核心参数详解：
    
    - `ensure_ascii`：默认`True`，非 ASCII 字符会被转义为`\uXXXX`，设置为`False`可保留中文等非 ASCII 字符，**处理中文时必须设置为 False**；
    - `indent`：缩进空格数，设置为 2/4 可格式化输出 JSON 字符串，提升可读性；
    - `sort_keys`：默认`False`，设置为`True`会按键名升序排序；
    - `default`：自定义序列化函数，用于处理 JSON 不支持的 Python 对象（如 datetime、自定义类）。
    
    示例：

    ```python
    import json
    from datetime import datetime
    
    data = {
        "name": "张三",
        "age": 18,
        "is_valid": True,
        "hobbies": ["Python", "Reading"],
        "create_time": datetime.now()
    }
    
    # 自定义序列化函数，处理datetime类型
    def datetime_serializer(obj):
        if isinstance(obj, datetime):
            return obj.strftime("%Y-%m-%d %H:%M:%S")
        raise TypeError(f"Object of type {obj.__class__.__name__} is not JSON serializable")
    
    # 序列化为格式化JSON字符串，保留中文
    json_str = json.dumps(data, ensure_ascii=False, indent=2, default=datetime_serializer)
    print(json_str)
    
    # 序列化写入文件
    with open("data.json", "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2, default=datetime_serializer)
    ```
    
2. **反序列化：JSON 字符串 / 文件 → Python 对象**

    |函数|功能描述|
    |---|---|
    |`json.loads(s, object_hook=None)`|将 JSON 字符串反序列化为 Python 对象|
    |`json.load(fp, object_hook=None)`|从文件对象 fp 中读取 JSON 字符串，反序列化为 Python 对象|
    
    核心参数`object_hook`：自定义反序列化函数，用于将 JSON 对象转换为自定义的 Python 类实例。
    
    示例：

    ```python
    import json
    
    # 从字符串反序列化
    json_str = '{"name": "张三", "age": 18, "is_valid": true}'
    data = json.loads(json_str)
    print(data["name"], data["age"])
    
    # 从文件反序列化
    with open("data.json", "r", encoding="utf-8") as f:
        data = json.load(f)
    ```
    

### 6.6.7 re 模块：正则表达式

`re`模块是 Python 的正则表达式处理模块，提供了字符串的模式匹配、查找、替换、分割等功能，是文本处理、数据提取、格式校验的核心工具。正则表达式的核心是通过元字符定义匹配规则，实现复杂的文本处理逻辑。

#### 1. 核心元字符与语法

|元字符|功能描述||
|---|---|---|
|`.`|匹配除换行符`\n`外的任意单个字符||
|`^`|匹配字符串的开头||
|`$`|匹配字符串的结尾||
|`*`|匹配前一个表达式 0 次或多次，贪婪匹配||
|`+`|匹配前一个表达式 1 次或多次，贪婪匹配||
|`?`|匹配前一个表达式 0 次或 1 次，贪婪匹配；跟在`*`/`+`后转为非贪婪匹配||
|`{n}`|匹配前一个表达式恰好 n 次||
|`{n,}`|匹配前一个表达式至少 n 次||
|`{n,m}`|匹配前一个表达式 n 到 m 次||
|`[]`|字符集，匹配括号内的任意一个字符，支持范围如`[a-z]`、`[0-9]`||
|`[^]`|否定字符集，匹配不在括号内的任意字符||
|`|`|或运算符，匹配左右任意一个表达式|
|`()`|分组，将括号内的表达式作为一个整体，捕获匹配的内容||
|`(?:)`|非捕获分组，仅作为整体匹配，不捕获内容||
|`\d`|匹配任意数字，等价于`[0-9]`||
|`\D`|匹配任意非数字，等价于`[^0-9]`||
|`\w`|匹配任意字母、数字、下划线，等价于`[a-zA-Z0-9_]`||
|`\W`|匹配任意非字母数字下划线，等价于`[^a-zA-Z0-9_]`||
|`\s`|匹配任意空白字符（空格、制表符、换行符等）||
|`\S`|匹配任意非空白字符||
|`\b`|匹配单词边界||
|`\B`|匹配非单词边界||

#### 2. 核心匹配模式

|模式常量|功能描述|
|---|---|
|`re.IGNORECASE` / `re.I`|忽略大小写匹配|
|`re.MULTILINE` / `re.M`|多行模式，`^`和`$`匹配每行的开头和结尾|
|`re.DOTALL` / `re.S`|让`.`匹配包括换行符`\n`在内的所有字符|
|`re.VERBOSE` / `re.X`|详细模式，忽略正则中的空格和注释，提升复杂正则的可读性|

#### 3. 高频核心函数

|函数|功能描述|核心特性|
|---|---|---|
|`re.match(pattern, string, flags=0)`|从字符串的**开头**匹配正则表达式|仅匹配开头，匹配成功返回匹配对象，失败返回 None|
|`re.search(pattern, string, flags=0)`|在整个字符串中搜索第一个匹配的位置|找到第一个匹配即返回，匹配成功返回匹配对象，失败返回 None|
|`re.findall(pattern, string, flags=0)`|查找字符串中所有匹配的内容，返回列表|无匹配返回空列表，有分组时返回分组内容的列表|
|`re.finditer(pattern, string, flags=0)`|查找字符串中所有匹配的内容，返回匹配对象的迭代器|适合大文本处理，惰性迭代，内存占用低|
|`re.sub(pattern, repl, string, count=0, flags=0)`|将字符串中匹配的内容替换为 repl，返回替换后的字符串|count 为最大替换次数，默认 0 替换所有，repl 支持分组引用|
|`re.split(pattern, string, maxsplit=0, flags=0)`|按正则匹配的内容分割字符串，返回分割后的列表|maxsplit 为最大分割次数，默认 0 全部分割|
|`re.compile(pattern, flags=0)`|将正则表达式编译为正则对象，提升重复匹配的性能|编译后的对象可复用，支持上述所有方法|

#### 4. 最佳实践

1. **预编译正则**：重复使用的正则表达式，使用`re.compile()`预编译，避免每次匹配都重新解析正则，大幅提升性能；
2. **使用原始字符串**：正则表达式使用`r''`原始字符串，避免转义字符的双重转义问题；
3. **非贪婪匹配**：默认`*`/`+`为贪婪匹配，会尽可能多的匹配内容，需在其后加`?`转为非贪婪匹配，实现精准匹配；
4. **分组捕获**：使用`()`分组提取需要的内容，无需捕获的分组使用`(?:)`减少内存开销；
5. **禁止过度复杂正则**：过于复杂的正则表达式可读性差、维护成本高，应拆分为多个简单正则，或使用其他文本处理方案。

示例：正则高频场景

```python
import re

# 1. 预编译正则：手机号校验
phone_pattern = re.compile(r"^1[3-9]\d{9}$")
# 校验手机号
print(phone_pattern.match("13812345678"))  # 返回匹配对象，有效
print(phone_pattern.match("12345678901"))  # 返回None，无效

# 2. 提取文本中的所有邮箱
text = "联系邮箱：zhangsan@163.com，lisi@gmail.com，wangwu@qq.com"
email_pattern = re.compile(r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}")
emails = email_pattern.findall(text)
print(emails)  # 输出：['zhangsan@163.com', 'lisi@gmail.com', 'wangwu@qq.com']

# 3. 替换文本中的敏感词
text = "这个产品太差了，垃圾，非常不好用"
sensitive_pattern = re.compile(r"垃圾|太差|不好")
new_text = sensitive_pattern.sub("***", text)
print(new_text)  # 输出：这个产品***了，***，非常***用

# 4. 非贪婪匹配提取HTML标签内容
html = "<div>标题</div><div>内容</div>"
# 非贪婪匹配
tag_pattern = re.compile(r"<div>(.*?)</div>")
contents = tag_pattern.findall(html)
print(contents)  # 输出：['标题', '内容']
```

### 6.6.8 collections 模块：扩展数据结构

`collections`模块提供了 Python 内置数据结构（list、dict、tuple、set）的扩展实现，补充了更高效、功能更丰富的容器数据类型，是工程化开发中提升代码效率、简化逻辑的核心工具。

高频核心数据结构：

1. **namedtuple：命名元组**
    
    用于创建带字段名的元组子类，可通过字段名访问元素，也可通过索引访问，兼顾元组的不可变性与可读性，替代普通元组与简单数据类。
    
    示例：

    ```python
    from collections import namedtuple
    
    # 定义命名元组类
    User = namedtuple("User", ["user_id", "name", "age", "gender"])
    # 创建实例
    user = User(user_id=1001, name="张三", age=18, gender="男")
    # 访问元素
    print(user.name)  # 字段名访问，输出：张三
    print(user[1])    # 索引访问，输出：张三
    # 转换为字典
    print(user._asdict())  # 输出：{'user_id': 1001, 'name': '张三', 'age': 18, 'gender': '男'}
    ```
    
2. **deque：双端队列**
    
    双端队列，实现了 O (1) 时间复杂度的两端追加与弹出操作，相比列表的头部操作（O (n)）性能提升显著，适用于队列、栈、消息队列、滑动窗口等场景。
    
    核心方法：

    ```python
    from collections import deque
    
    # 创建双端队列
    dq = deque([1, 2, 3], maxlen=10)  # maxlen设置最大长度，超出自动丢弃另一端元素
    
    # 两端操作，O(1)时间复杂度
    dq.append(4)       # 尾部追加
    dq.appendleft(0)   # 头部追加
    dq.pop()           # 尾部弹出
    dq.popleft()       # 头部弹出
    ```
    
3. **defaultdict：默认值字典**
    
    字典的子类，访问不存在的键时，会自动创建该键，并为其设置指定类型的默认值，避免`KeyError`异常，无需提前判断键是否存在，适用于数据统计、分组等场景。
    
    示例：

    ```python
    from collections import defaultdict
    
    # 定义默认值为列表的字典
    group_dict = defaultdict(list)
    # 无需判断键是否存在，直接append
    group_dict["男"].append("张三")
    group_dict["女"].append("李四")
    group_dict["男"].append("王五")
    print(group_dict)  # 输出：defaultdict(<class 'list'>, {'男': ['张三', '王五'], '女': ['李四']})
    
    # 计数场景，默认值为0
    count_dict = defaultdict(int)
    for word in ["a", "b", "a", "c", "b", "a"]:
        count_dict[word] += 1
    print(count_dict)  # 输出：defaultdict(<class 'int'>, {'a': 3, 'b': 2, 'c': 1})
    ```
    
4. **Counter：计数器**
    
    字典的子类，专门用于可迭代对象的元素计数，提供了计数统计、排序、TopN、集合运算等高频功能，是数据统计、词频统计的首选工具。
    
    示例：

    ```python
    from collections import Counter
    
    # 词频统计
    words = ["python", "java", "python", "c++", "python", "java", "go"]
    word_count = Counter(words)
    print(word_count)  # 输出：Counter({'python': 3, 'java': 2, 'c++': 1, 'go': 1})
    
    # 获取Top2高频元素
    print(word_count.most_common(2))  # 输出：[('python', 3), ('java', 2)]
    
    # 元素总数
    print(sum(word_count.values()))  # 输出：7
    
    # 集合运算
    other_count = Counter({"python": 2, "java": 1, "php": 2})
    print(word_count + other_count)  # 并集计数相加
    print(word_count - other_count)  # 差集计数相减
    ```
    
5. **OrderedDict：有序字典**
    
    字典的子类，保留键值对的插入顺序，Python 3.7 + 内置字典已默认保留插入顺序，该类的核心优势是提供了顺序相关的专用方法，如`move_to_end()`、`popitem(last=True)`等，适用于需要精细控制顺序的场景，如 LRU 缓存实现。
    
6. **ChainMap：链式映射**
    
    用于将多个字典 / 映射对象组合为一个逻辑上的整体，无需合并字典，节省内存，查找时按顺序遍历所有映射，找到第一个匹配的键即返回，适用于多配置层级、多命名空间合并的场景。
    

---

## 本章核心考点与学习要求

1. 深刻理解模块与包的定义、import 机制的底层执行流程，熟练使用绝对导入与相对导入，掌握`__init__.py`的核心作用，解决模块导入的常见报错，遵循 PEP8 导入规范；
2. 彻底掌握`__name__ == '__main__'`的底层原理与工程意义，规范编写 Python 程序入口，实现模块的双重身份；
3. 熟练掌握 Python 异常体系，完整掌握`try/except/else/finally`的执行规则，精准捕获异常，遵循异常处理最佳实践，避免常见的异常处理坑点；
4. 熟练使用`raise`关键字主动抛出异常，规范实现自定义业务异常体系，掌握异常链的用法，实现精细化的异常处理；
5. 彻底掌握文件操作的完整语法，熟练使用`open()`函数的各类模式，理解文本模式与二进制模式的区别，强制遵循编码规范，熟练使用`with`语句管理文件资源，掌握大文件的流式读取最佳实践；
6. 熟练掌握本章讲解的 Python 标准库核心模块的高频用法，能根据业务场景选择合适的标准库工具，写出简洁、高效、符合 Python 规范的代码；
7. 掌握正则表达式的核心语法与高频函数，能实现文本校验、数据提取、替换等常见文本处理场景，遵循正则最佳实践；
8. 能将本章内容应用到工程化开发中，实现规范的代码组织、健壮的异常容错、安全的文件操作、高效的数据处理，解决相关的面试高频考点。