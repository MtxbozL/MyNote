
迭代器是 Python for 循环的底层实现核心，是 Python 惰性计算体系的基础，遵循**迭代器协议**，是实现高效、低内存数据处理的核心工具。

### 5.3.1 迭代器协议核心定义

Python 的迭代器体系基于两个核心抽象，均定义在`collections.abc`模块中：

1. **可迭代对象（Iterable）**
    
    实现了`__iter__()`方法的对象，即为可迭代对象。`__iter__()`方法必须返回一个迭代器对象。
    
    Python 中所有序列类型（列表、元组、字符串）、字典、集合、文件对象、生成器均为可迭代对象，所有可迭代对象均可用于 for 循环遍历。
    
2. **迭代器对象（Iterator）**
    
    同时实现了`__iter__()`和`__next__()`方法的对象，即为迭代器对象，严格遵循迭代器协议：
    
    - `__iter__()`方法：返回迭代器自身，保证迭代器也是可迭代对象，可用于 for 循环；
    - `__next__()`方法：返回迭代器的下一个元素，当迭代器无元素可返回时，必须抛出`StopIteration`异常，终止迭代。
    

### 5.3.2 可迭代对象与迭代器的核心区别

|特性|可迭代对象（Iterable）|迭代器对象（Iterator）|
|---|---|---|
|核心方法|仅实现`__iter__()`|同时实现`__iter__()`和`__next__()`|
|核心功能|提供迭代器，用于遍历|执行迭代，生成下一个元素|
|遍历特性|可多次遍历，每次遍历生成新的迭代器|仅可一次性遍历，遍历结束后无法重置|
|内存特性|存储完整数据，内存占用与数据量正相关|不存储完整数据，仅保留迭代状态，内存固定|
|示例|列表、元组、字典、字符串|生成器、文件迭代器、iter () 返回的对象|

### 5.3.3 iter () 与 next () 内置函数

二者是迭代器协议的官方调用入口，是 for 循环的底层依赖：

1. **iter(iterable)**：调用可迭代对象的`__iter__()`方法，返回对应的迭代器对象；若传入的是迭代器，则返回其自身。
2. **next(iterator)**：调用迭代器的`__next__()`方法，返回下一个元素；当迭代器耗尽时，抛出`StopIteration`异常。

示例：迭代器的基础使用

```python
# 可迭代对象：列表
nums = [1, 2, 3]
# 获取迭代器对象
it = iter(nums)

# 手动迭代
print(next(it))  # 输出：1
print(next(it))  # 输出：2
print(next(it))  # 输出：3
# 迭代器耗尽，抛出StopIteration
# print(next(it))
```

### 5.3.4 Python for 循环的底层执行原理

Python 的 for 循环本质是**迭代器的语法糖**，其底层执行流程完全基于迭代器协议，与以下 while 循环完全等价：

```python
# 原生for循环
nums = [1, 2, 3]
for num in nums:
    print(num)

# 底层等价实现
it = iter(nums)
while True:
    try:
        num = next(it)
        print(num)
    except StopIteration:
        # 捕获终止异常，退出循环
        break
```

核心流程：

1. 调用`iter()`获取可迭代对象的迭代器；
2. 循环调用`next()`获取下一个元素，执行循环体；
3. 捕获`StopIteration`异常，终止循环，不会向外抛出。

这也是为什么 Python 中所有可迭代对象都可用于 for 循环的核心原因 —— 只要实现了迭代器协议，就可被 for 循环统一处理。

### 5.3.5 迭代器的核心特性

1. **惰性计算**：迭代器不会一次性生成所有元素，仅当调用`next()`时才会生成下一个元素，无需提前加载所有数据到内存，是处理超大数据、无限序列的核心方案。
2. **一次性消耗**：迭代器仅可遍历一次，遍历结束后，再次调用`next()`会持续抛出`StopIteration`异常，无法重置、无法回头。
3. **内存高效**：迭代器本身不存储完整数据，仅保留当前迭代位置与生成规则，无论数据量多大，内存占用始终固定，不会随数据量增长而增加。

### 5.3.6 自定义迭代器

通过实现迭代器协议，可自定义迭代器类，实现任意规则的迭代逻辑，核心是实现`__iter__()`和`__next__()`两个方法。

示例：斐波那契数列迭代器，生成无限序列

```python
class FibIterator:
    def __init__(self, max_num=None):
        # 初始化迭代器状态
        self.a, self.b = 0, 1
        self.max_num = max_num  # 可选最大值限制
    
    def __iter__(self):
        # 返回迭代器自身
        return self
    
    def __next__(self):
        # 生成下一个元素，终止时抛出StopIteration
        if self.max_num is not None and self.a > self.max_num:
            raise StopIteration
        current = self.a
        self.a, self.b = self.b, self.a + self.b
        return current

# 使用自定义迭代器
fib = FibIterator(100)
for num in fib:
    print(num, end=" ")
# 输出：0 1 1 2 3 5 8 13 21 34 55 89
```

---
