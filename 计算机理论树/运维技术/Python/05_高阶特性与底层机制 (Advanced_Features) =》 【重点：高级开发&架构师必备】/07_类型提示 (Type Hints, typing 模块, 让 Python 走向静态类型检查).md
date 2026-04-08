
类型提示是 Python 3.5 引入的核心特性（PEP 484 规范），为动态类型的 Python 提供了静态类型注解能力，是现代 Python 工程化、大型项目开发、团队协作的必备规范。其核心作用是提升代码可读性、可维护性，配合静态类型检查工具，在运行前发现类型错误，减少线上 bug。

### 5.7.1 类型提示的核心语法

Python 解释器会完全忽略类型注解，不会影响代码的运行，仅用于开发阶段的静态检查、IDE 智能补全与文档生成。

#### 1. 基础类型注解

- 变量注解：`变量名: 类型 = 值`
- 函数参数与返回值注解：`def 函数名(参数名: 类型) -> 返回值类型:`

示例：

```python
# 基础变量注解
name: str = "Python"
version: float = 3.12
is_valid: bool = True

# 函数参数与返回值注解
def add(a: int, b: int) -> int:
    """两个整数相加，返回整数结果"""
    return a + b

# 无返回值的函数，返回值类型为None
def log_message(msg: str) -> None:
    print(f"[LOG] {msg}")
```

#### 2. 复合类型注解

Python 3.9 + 支持内置容器类型的泛型注解（PEP 585 规范），无需从`typing`模块导入`List`/`Dict`等；Python 3.8 及以下版本需从`typing`模块导入对应类型。

核心复合类型注解如下表：

|场景|Python 3.9+ 语法|低版本兼容语法（typing）|功能描述|
|---|---|---|---|
|列表|`list[T]`|`List[T]`|元素类型为 T 的列表|
|固定长度元组|`tuple[T1, T2, T3]`|`Tuple[T1, T2, T3]`|固定长度、对应位置类型固定的元组|
|可变长度元组|`tuple[T, ...]`|`Tuple[T, ...]`|任意长度、所有元素类型均为 T 的元组|
|字典|`dict[K, V]`|`Dict[K, V]`|键类型为 K、值类型为 V 的字典|
|集合|`set[T]`|`Set[T]`|元素类型为 T 的集合|
|可选类型|`T \| None`|`Optional[T]`|类型可以是 T，也可以是 None|
|联合类型|`T1 \| T2 \| T3`|`Union[T1, T2, T3]`|类型可以是指定类型中的任意一种|
|可调用对象|`Callable[[P1, P2], R]`|`Callable[[P1, P2], R]`|接收 P1/P2 类型参数，返回 R 类型的可调用对象|
|任意类型|-|`Any`|关闭类型检查，兼容所有类型，尽量少用|
|无返回|-|`NoReturn`|函数永远不会正常返回（如一定会抛出异常）|
|字面量类型|`Literal["v1", "v2"]`|`Literal["v1", "v2"]`|仅允许指定的字面量值|

示例：复合类型注解

```python
# Python 3.10+ 语法
from typing import Callable, Literal, NoReturn

# 列表注解
nums: list[int] = [1, 2, 3, 4]

# 字典注解
user_info: dict[str, str | int] = {"name": "张三", "age": 18}

# 可选类型注解
def get_user(user_id: int) -> dict[str, str | int] | None:
    """根据用户ID获取用户信息，不存在则返回None"""
    pass

# 可调用对象注解
def apply_func(func: Callable[[int, int], int], a: int, b: int) -> int:
    return func(a, b)

# 字面量类型注解
Gender = Literal["男", "女", "其他"]
def set_gender(gender: Gender) -> None:
    pass

# NoReturn注解
def raise_error(msg: str) -> NoReturn:
    raise ValueError(msg)
```

### 5.7.2 进阶类型注解用法

1. **类型别名**：为复杂的类型定义别名，提升代码可读性，Python 3.10 + 支持`TypeAlias`显式声明。
    
    示例：

    ```python
    # 类型别名
    UserDict = dict[str, str | int]
    UserList = list[UserDict]
    
    def get_users() -> UserList:
        return [{"name": "张三", "age": 18}]
    ```
    
2. **泛型注解**：通过`TypeVar`定义泛型类型变量，实现通用函数 / 类的类型注解，适配多种类型。
    
    示例：

    ```python
    from typing import TypeVar
    
    # 定义泛型类型变量
    T = TypeVar("T")
    
    def get_first_item(lst: list[T]) -> T | None:
        """接收元素类型为T的列表，返回第一个元素（类型为T）或None"""
        return lst[0] if lst else None
    ```
    
3. **自定义类作为类型**：自定义的类可直接作为类型注解，IDE 会自动识别类的属性与方法。
    
    示例：

    ```python
    class User:
        def __init__(self, name: str, age: int):
            self.name = name
            self.age = age
    
    def create_user(name: str, age: int) -> User:
        return User(name, age)
    ```
    

### 5.7.3 静态类型检查工具

类型注解的核心价值通过静态类型检查工具实现，主流工具包括：

1. **mypy**：Python 官方推荐的静态类型检查器，是行业事实标准；
2. **pyright**：微软开发的高性能类型检查器，VSCode 的 Python 扩展内置；
3. **pytype**：谷歌开发的类型检查器，支持类型推断。

使用方式：安装后，通过命令行执行`mypy 文件名.py`，即可对代码进行静态类型检查，提前发现类型不匹配的错误。

### 5.7.4 类型提示的核心价值

1. **提升代码可读性**：无需阅读函数实现，即可通过注解明确参数与返回值的类型，降低维护成本；
2. **IDE 智能补全**：IDE 可基于类型注解提供精准的智能补全、语法提示，大幅提升开发效率；
3. **提前发现 bug**：静态类型检查可在代码运行前发现类型不匹配的错误，减少线上 bug；
4. **团队协作规范**：大型项目、团队协作中，类型提示是强制代码规范、降低沟通成本的核心工具；
5. **框架支持**：现代 Python 框架（FastAPI、Django 4.0+、Pydantic 等）均深度依赖类型提示，实现数据校验、自动文档生成等核心功能。

---

## 本章核心考点与学习要求

1. 深刻理解闭包的定义、形成条件与底层实现，彻底掌握延迟绑定陷阱的原理与解决方案，熟练使用闭包实现状态保持与函数定制；
2. 彻底掌握装饰器的底层本质，熟练实现基础无参装饰器、带参数装饰器、类装饰器，深刻理解`functools.wraps`的核心作用，掌握多装饰器的执行顺序，能实现日志、计时、缓存等通用装饰器；
3. 深刻理解迭代器协议，明确可迭代对象与迭代器的核心区别，彻底掌握 Python for 循环的底层执行原理，能自定义实现迭代器类；
4. 熟练掌握生成器函数与生成器表达式的实现，深刻理解`yield`关键字的执行原理，熟练使用`yield from`实现生成器委托，掌握生成器的惰性计算特性与内存优势；
5. 深刻理解上下文管理器协议，熟练通过类与`contextlib`模块实现上下文管理器，掌握`with`语句的底层执行流程，能实现资源管理的上下文管理器；
6. 彻底掌握 Python 垃圾回收机制的三层架构，深刻理解引用计数的原理、循环引用的问题，掌握标记 - 清除与分代回收的核心原理，熟悉`gc`模块的核心用法；
7. 熟练掌握 Python 类型提示的完整语法，能为变量、函数、复合类型编写规范的类型注解，了解静态类型检查工具的使用，理解类型提示在工程化开发中的核心价值；
8. 能将本章高阶特性应用到工程化开发中，写出符合 Pythonic 规范、高可维护性、低内存占用的代码，解决相关的高频面试题。