
装饰器是闭包的经典工程化应用，是 Python 的核心语法糖（PEP 318 规范），核心设计思想遵循**开闭原则**：对扩展开放，对修改关闭。其核心作用是：**在不修改原函数的源代码、不改变原函数的调用方式的前提下，为原函数动态添加额外的通用功能**。

### 5.2.1 装饰器的底层本质

Python 中`@`装饰器语法糖的本质，是函数的重新赋值：

```python
@decorator
def target_func():
    pass
```

上述代码完全等价于：

```python
def target_func():
    pass
target_func = decorator(target_func)
```

核心逻辑：将原函数作为参数传入装饰器函数，装饰器返回一个新的包装函数，并用原函数的变量名接收这个新函数，实现对原函数的无侵入式替换。

### 5.2.2 基础无参函数装饰器

基础装饰器为两层嵌套的闭包结构，外层函数接收原函数作为参数，内层包装函数实现额外功能 + 调用原函数，最终返回内层包装函数。

示例：通用函数计时装饰器

```python
import time
import functools

def timer(func):
    # 外层函数：接收原函数func，返回包装函数wrapper
    @functools.wraps(func)  # 保留原函数元信息，5.2.3节详解
    def wrapper(*args, **kwargs):
        # 内层包装函数：实现额外功能，接收原函数的所有参数
        start_time = time.perf_counter()
        # 调用原函数，接收返回值，兼容所有参数类型与返回值
        result = func(*args, **kwargs)
        end_time = time.perf_counter()
        print(f"函数 {func.__name__} 执行耗时：{end_time - start_time:.6f}s")
        # 返回原函数的执行结果，保证原函数功能完整
        return result
    # 返回包装函数，替换原函数
    return wrapper

# 使用装饰器
@timer
def calculate_sum(n):
    return sum(range(n))

# 原函数调用方式完全不变
print(calculate_sum(1000000))
```

核心执行流程：

1. 解释器执行`@timer`时，将`calculate_sum`函数传入`timer`，执行外层函数，返回`wrapper`函数；
2. 用`calculate_sum`变量名接收`wrapper`函数，后续所有对`calculate_sum`的调用，实际都是调用`wrapper`函数；
3. `wrapper`函数接收原函数的所有参数，执行额外功能，调用原函数，返回原函数的执行结果，保证原函数功能完全不受影响。

### 5.2.3 functools.wraps 的核心作用

装饰器的核心陷阱：原函数被装饰器替换后，其元信息（函数名`__name__`、文档字符串`__doc__`、模块名`__module__`、参数签名等）会被包装函数`wrapper`覆盖，导致调试、文档生成、序列化等场景出现非预期问题。

`functools.wraps`装饰器的核心作用：**将原函数的所有元信息拷贝到包装函数中，保留原函数的元数据特征**，是实现装饰器的强制规范。

反例（未使用 wraps）：

```python
def timer(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@timer
def target_func():
    """这是目标函数的文档字符串"""
    pass

print(target_func.__name__)  # 输出：wrapper，而非target_func
print(target_func.__doc__)   # 输出：None，原文档字符串丢失
```

### 5.2.4 带参数的装饰器

带参数的装饰器本质是**三层嵌套的闭包结构**，因为装饰器语法要求`@decorator(args)`必须返回一个标准的无参装饰器函数，其等价逻辑为：

```python
@decorator(arg1, arg2)
def target_func():
    pass
```

完全等价于：

```python
def target_func():
    pass
target_func = decorator(arg1, arg2)(target_func)
```

三层结构分工：

1. 第一层函数：接收装饰器的自定义参数，返回第二层的标准装饰器函数；
2. 第二层函数：接收原函数，返回第三层的包装函数；
3. 第三层函数：实现额外功能与原函数调用，与基础装饰器的 wrapper 一致。

示例：带日志级别参数的日志装饰器

```python
import functools
import logging

def logger(level="INFO"):
    # 第一层：接收装饰器参数，返回标准装饰器
    def decorator(func):
        # 第二层：标准装饰器，接收原函数
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            # 第三层：包装函数，使用外层的level参数
            log_msg = f"执行函数：{func.__name__}，参数：{args} {kwargs}"
            if level == "DEBUG":
                logging.debug(log_msg)
            elif level == "INFO":
                logging.info(log_msg)
            elif level == "ERROR":
                logging.error(log_msg)
            return func(*args, **kwargs)
        return wrapper
    return decorator

# 使用带参数的装饰器
@logger(level="DEBUG")
def add(a, b):
    return a + b

add(3, 4)
```

### 5.2.5 类装饰器

类装饰器是通过类实现的装饰器，核心依赖第四章讲解的`__call__`魔术方法：实现了`__call__`方法的类，其实例是可调用对象，可作为装饰器使用。

类装饰器的核心优势：通过类的属性维护状态，比函数闭包的状态管理更清晰、更易扩展，适合实现复杂的、带状态的装饰器（如调用次数统计、缓存、限流等）。

基础语法与执行逻辑：

```python
@Decorator
def target_func():
    pass
```

等价于：

```python
def target_func():
    pass
target_func = Decorator(target_func)
```

执行流程：`@`语法会实例化 Decorator 类，将原函数传入`__init__`方法，后续调用原函数时，会触发实例的`__call__`方法。

示例：函数调用次数统计装饰器

```python
import functools

class Counter:
    def __init__(self, func):
        # 初始化时接收原函数，保存到实例属性
        self.func = func
        # 类属性维护调用次数状态
        self.call_count = 0
        # 保留原函数元信息
        functools.wraps(func)(self)
    
    def __call__(self, *args, **kwargs):
        # 函数被调用时触发，实现装饰器逻辑
        self.call_count += 1
        print(f"函数 {self.func.__name__} 已被调用 {self.call_count} 次")
        return self.func(*args, **kwargs)

# 使用类装饰器
@Counter
def target_func():
    print("函数执行")

target_func()  # 输出：已被调用1次
target_func()  # 输出：已被调用2次
print(target_func.call_count)  # 输出：2
```

### 5.2.6 装饰器进阶规则

1. **多装饰器执行顺序**：多个装饰器装饰同一个函数时，**装饰顺序从上到下，执行顺序从下到上**（离函数最近的装饰器先执行）。
    
    示例：

    ```python
    @decorator1
    @decorator2
    def func():
        pass
    ```
    
    等价于`func = decorator1(decorator2(func))`，执行时先执行 decorator2 的 wrapper，再执行 decorator1 的 wrapper。
2. **内置通用装饰器**：Python 内置了多个高频装饰器，如`functools.lru_cache`（LRU 缓存装饰器，实现函数结果缓存）、`functools.total_ordering`（补全类的比较运算符）、`@property`（第四章已讲解）。
3. **装饰器适用场景**：日志记录、性能计时、权限校验、参数校验、结果缓存、重试机制、接口限流、事务管理等通用横切逻辑。

---
