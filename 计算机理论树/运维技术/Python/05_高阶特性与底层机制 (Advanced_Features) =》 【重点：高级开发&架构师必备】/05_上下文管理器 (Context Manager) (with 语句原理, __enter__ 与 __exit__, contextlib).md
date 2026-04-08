
上下文管理器是 Python 用于**资源安全管理**的核心协议（PEP 343 规范），核心作用是保证资源的获取与释放逻辑一定会执行，无论代码正常执行还是发生异常，彻底避免资源泄漏问题，是文件操作、数据库连接、锁管理等场景的标准实现方案。

### 5.5.1 with 语句的核心本质

上下文管理器通过`with`语句使用，替代了传统的`try/except/finally`资源释放模式，语法更简洁、逻辑更安全。

基础语法：

```python
with 上下文管理器 as 资源变量:
    # 代码块，使用资源
    执行语句
# 离开with块时，自动释放资源
```

`with`语句的底层执行流程：

1. 执行上下文管理器表达式，获取上下文管理器对象；
2. 调用上下文管理器的`__enter__()`方法，返回值赋值给`as`后的变量；
3. 执行`with`块内的代码；
4. 无论代码正常执行结束、还是发生异常，都会调用上下文管理器的`__exit__()`方法，执行资源释放、清理逻辑。

### 5.5.2 上下文管理器协议：**enter** 与 **exit**

一个类只要实现了`__enter__()`和`__exit__()`两个魔术方法，其实例就是上下文管理器，严格遵循上下文管理器协议。

两个方法的详细规范：

1. ****enter**(self)**
    
    - 进入`with`代码块时自动调用；
    - 返回值会赋值给`as`后的变量，通常返回需要管理的资源对象；
    - 可在该方法中实现资源的获取、初始化逻辑。
    
2. ****exit**(self, exc_type, exc_val, exc_tb)**
    
    - 离开`with`代码块时自动调用，无论是否发生异常都会执行，是资源释放的核心位置；
    - 三个参数用于接收异常信息：
        
        - `exc_type`：异常类型，无异常时为`None`；
        - `exc_val`：异常实例，无异常时为`None`；
        - `exc_tb`：异常回溯对象，无异常时为`None`；
        
    - 返回值规则：若返回`True`，则抑制`with`块中抛出的异常，不会向外传播；若返回`False`或无返回值，异常会继续向外抛出。
    

示例：自定义文件操作上下文管理器

```python
class FileHandler:
    def __init__(self, file_path, mode="r", encoding="utf-8"):
        self.file_path = file_path
        self.mode = mode
        self.encoding = encoding
        self.file_obj = None

    def __enter__(self):
        # 进入with块时，打开文件，返回文件对象
        self.file_obj = open(self.file_path, self.mode, encoding=self.encoding)
        return self.file_obj

    def __exit__(self, exc_type, exc_val, exc_tb):
        # 离开with块时，关闭文件，无论是否异常都会执行
        if self.file_obj:
            self.file_obj.close()
        # 异常不抑制，继续向外传播
        return False

# 使用自定义上下文管理器
with FileHandler("test.txt", "w") as f:
    f.write("Hello Python")
# 离开with块，文件已自动关闭
```

### 5.5.3 contextlib 模块：简化上下文管理器实现

`contextlib`是 Python 内置的上下文管理器工具模块，提供了极简的实现方式，无需定义类与两个魔术方法，通过装饰器 + 生成器即可实现上下文管理器，是轻量级上下文管理器的首选方案。

核心工具：`@contextlib.contextmanager`装饰器

- 装饰一个生成器函数，将其转换为上下文管理器；
- 生成器函数必须**仅 yield 一次**，`yield`之前的代码对应`__enter__()`方法，在进入`with`块时执行；`yield`之后的代码对应`__exit__()`方法，在离开`with`块时执行；
- `yield`的返回值对应`as`后的变量；
- 必须通过`try/finally`保证`yield`后的清理代码一定会执行，即使发生异常。

示例：用 contextmanager 实现计时上下文管理器

```python
import time
from contextlib import contextmanager

@contextmanager
def timer():
    # __enter__逻辑：进入with块时执行
    start_time = time.perf_counter()
    try:
        # yield返回值赋值给as后的变量，暂停执行
        yield
    finally:
        # __exit__逻辑：离开with块时执行，无论是否异常都会执行
        end_time = time.perf_counter()
        print(f"代码块执行耗时：{end_time - start_time:.6f}s")

# 使用
with timer():
    # 待计时的代码块
    sum(range(1000000))
# 离开with块，自动打印耗时
```

`contextlib`模块其他高频工具：

1. `closing(obj)`：自动为带有`close()`方法的对象生成上下文管理器，离开时自动调用`close()`方法；
2. `suppress(*exceptions)`：抑制指定的异常，`with`块中抛出指定异常时，不会向外传播；
3. `ExitStack`：管理多个上下文管理器的嵌套，动态批量管理资源的获取与释放。

### 5.5.4 适用场景

上下文管理器是所有资源管理场景的标准方案，高频应用场景包括：

- 文件 IO 操作（Python 内置`open()`函数已实现上下文管理器）；
- 数据库连接、Redis 连接、网络套接字等网络资源管理；
- 线程 / 进程锁的获取与释放，保证锁一定会被释放，避免死锁；
- 代码执行环境的临时切换（如临时修改工作目录、临时修改环境变量）；
- 事务管理（如数据库事务的提交与回滚）。

---
