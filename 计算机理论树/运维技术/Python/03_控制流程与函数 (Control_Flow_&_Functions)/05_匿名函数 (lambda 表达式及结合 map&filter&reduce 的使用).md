
匿名函数是无需通过`def`定义命名函数的单行函数，通过`lambda`关键字创建，核心用于临时、简单的函数逻辑，通常作为高阶函数的参数传入，避免定义大量仅使用一次的单行命名函数。

### 3.5.1 lambda 核心语法

```python
lambda 参数1, 参数2, ... : 表达式
```

核心规则：

1. lambda 函数的主体只能是**单个表达式**，不能包含循环、多分支代码块（仅支持三元表达式）、return 语句，表达式的计算结果就是函数的返回值；
2. 支持所有参数类型：位置参数、默认参数、*args、**kwargs、仅限关键字参数；
3. lambda 函数本质是一个函数对象，可赋值给变量，也可直接调用，示例：

    ```python
    # 赋值给变量，等价于def add(a, b): return a + b
    add = lambda a, b: a + b
    print(add(3, 4))  # 输出：7
    
    # 直接调用
    print((lambda x: x ** 2)(5))  # 输出：25
    ```
    

### 3.5.2 lambda 与高阶函数的结合使用

lambda 的核心适用场景是作为`sorted()`、`map()`、`filter()`、`reduce()`等高阶函数的 key/func 参数，实现简洁的内联函数逻辑。

1. **sorted () 排序场景（最常用）**
    
    sorted 函数的`key`参数接收一个函数，用于生成排序的键值，lambda 是该场景的首选，示例：

    ```python
    # 按元组的第二个元素排序
    students = [("张三", 90), ("李四", 85), ("王五", 95)]
    sorted_students = sorted(students, key=lambda x: x[1], reverse=True)
    print(sorted_students)  # 输出：[('王五', 95), ('张三', 90), ('李四', 85)]
    
    # 按字典的指定key排序
    user_list = [{"name": "Python", "version": 3.12}, {"name": "Java", "version": 17}]
    sorted_users = sorted(user_list, key=lambda x: x["version"])
    ```
    
2. **map () 函数**
    
    map 函数用于将一个函数应用到可迭代对象的每个元素，返回一个迭代器，语法：`map(func, *iterables)`，示例：

    ```python
    # 计算每个元素的平方
    nums = [1, 2, 3, 4]
    square_iter = map(lambda x: x ** 2, nums)
    print(list(square_iter))  # 输出：[1, 4, 9, 16]
    ```
    
    注意：Python3 中 map 返回迭代器，需转换为列表 / 元组才能查看所有元素；简单场景推荐使用列表推导式替代 map，可读性更高。
    
3. **filter () 函数**
    
    filter 函数用于过滤可迭代对象，保留 func 返回 True 的元素，返回一个迭代器，语法：`filter(func, iterable)`，示例：

    ```python
    # 过滤出偶数
    nums = [1, 2, 3, 4, 5, 6]
    even_iter = filter(lambda x: x % 2 == 0, nums)
    print(list(even_iter))  # 输出：[2, 4, 6]
    ```
    
    简单场景推荐使用列表推导式替代 filter。
    
4. **reduce () 函数**
    
    reduce 函数位于`functools`模块，用于对可迭代对象的元素进行累积计算，返回单个值，语法：`reduce(func, iterable[, initializer])`，func 必须接收两个参数，示例：

    ```python
    from functools import reduce
    
    # 计算列表所有元素的乘积
    nums = [1, 2, 3, 4]
    product = reduce(lambda a, b: a * b, nums)
    print(product)  # 输出：24
    ```
    

### 3.5.3 使用规范

遵循 PEP8 规范与 Python 最佳实践：

1. lambda 仅适用于单行、简单的函数逻辑，复杂逻辑必须使用 def 定义命名函数，禁止滥用 lambda 降低代码可读性；
2. 禁止将 lambda 赋值给变量用于复杂场景，该用法违背了匿名函数的设计初衷，且不利于异常栈调试；
3. 优先使用列表推导式替代复杂的 map/filter+lambda 组合，提升代码可读性。

---
