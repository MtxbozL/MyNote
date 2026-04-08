
生成器是 Python 中创建迭代器最简洁、最高效的方式，是**特殊的迭代器**，无需手动实现`__iter__()`和`__next__()`方法，仅需通过`yield`关键字即可自动实现迭代器协议，是 Python 惰性计算的核心载体（PEP 255 规范）。

### 5.4.1 生成器的创建方式

Python 提供两种创建生成器的方式：生成器函数与生成器表达式。

#### 1. 生成器函数

在函数中使用`yield`关键字替代`return`，该函数即为生成器函数。其核心特性：

- 调用生成器函数，**不会执行函数体代码**，仅返回一个生成器对象（迭代器）；
- 仅当调用`next()`或 for 循环遍历生成器时，才会执行函数体代码；
- 执行到`yield`语句时，函数会暂停执行，返回`yield`后的值，同时保存当前执行状态；
- 下次调用`next()`时，函数从上次暂停的位置继续执行，直到遇到下一个`yield`或函数结束。

示例：斐波那契数列生成器函数

```python
def fib_generator(max_num):
    a, b = 0, 1
    while a <= max_num:
        # 暂停执行，返回当前值，保存状态
        yield a
        a, b = b, a + b

# 调用生成器函数，返回生成器对象，不执行函数体
fib = fib_generator(100)
# 生成器是迭代器，可用于for循环
for num in fib:
    print(num, end=" ")
# 输出：0 1 1 2 3 5 8 13 21 34 55 89
```

#### 2. 生成器表达式

将列表推导式的方括号`[]`替换为圆括号`()`，即为生成器表达式，返回一个生成器对象，是创建简单生成器的极简方式。

示例：

```python
# 列表推导式：一次性生成所有元素，占用完整内存
list_comp = [i**2 for i in range(1000000)]
print(type(list_comp))  # 输出：<class 'list'>

# 生成器表达式：惰性生成，内存占用固定
gen_comp = (i**2 for i in range(1000000))
print(type(gen_comp))  # 输出：<class 'generator'>

# 遍历生成器，逐个生成元素
for num in gen_comp:
    if num > 100:
        break
    print(num, end=" ")
```

核心优势：生成器表达式无需提前生成所有元素，即使处理千万级数据，内存占用始终为固定值，远优于列表推导式。

### 5.4.2 yield 关键字的执行原理

`yield`是生成器的核心，其与`return`的本质区别如下：

|特性|yield|return|
|---|---|---|
|执行效果|暂停函数执行，保存当前状态，下次调用从暂停处继续|终止函数执行，销毁函数作用域，无法恢复|
|返回值|每次执行返回单个值，可多次返回|执行一次返回一个值（可多个打包为元组）|
|适用场景|生成器函数，迭代器实现|普通函数，函数结果返回|

生成器函数中`return`语句的规则：

- 生成器函数中的`return`语句会终止生成器，触发`StopIteration`异常；
- `return`后的值会作为`StopIteration`异常的`value`属性，不会直接返回给调用者；
- Python 3.3 + 支持生成器函数的`return`语句，低版本会触发语法错误。

### 5.4.3 yield from 语法

`yield from`是 Python 3.3 引入的语法（PEP 380 规范），核心作用是**在生成器中委托另一个可迭代对象 / 生成器**，简化嵌套生成器的代码，自动处理子生成器的`StopIteration`异常、返回值传递，是 Python 协程的底层实现基础。

核心用法分为两类：

1. **简化可迭代对象的遍历**
    
    `yield from iterable`完全等价于`for item in iterable: yield item`，大幅简化嵌套遍历代码。
    
    示例：嵌套列表展开

    ```python
    # 普通yield实现嵌套展开
    def flatten_nested(nested_list):
        for item in nested_list:
            if isinstance(item, list):
                for sub_item in flatten_nested(item):
                    yield sub_item
            else:
                yield item
    
    # yield from简化实现
    def flatten_nested(nested_list):
        for item in nested_list:
            if isinstance(item, list):
                # 委托子生成器，自动遍历并yield
                yield from flatten_nested(item)
            else:
                yield item
    
    # 测试
    nested = [1, [2, [3, 4], 5], 6]
    print(list(flatten_nested(nested)))  # 输出：[1, 2, 3, 4, 5, 6]
    ```
    
2. **生成器委托与返回值传递**
    
    `yield from`会自动处理子生成器的`next()`调用、异常捕获，同时可接收子生成器`return`的返回值，无需手动捕获`StopIteration`异常，是复杂生成器嵌套的标准实现方案。
    

### 5.4.4 生成器的核心优势与适用场景

1. **极致的内存效率**：惰性计算，按需生成元素，无需提前加载全量数据，是处理 GB 级超大文件、超大数据集的首选方案；
2. **代码简洁性**：相比自定义迭代器，无需实现复杂的类与协议方法，仅需`yield`关键字即可实现迭代逻辑；
3. **无限序列支持**：可实现无限序列的生成，无需担心内存溢出；
4. **协程实现基础**：Python 早期的协程完全基于生成器实现，是现代异步编程的底层核心。

适用场景：超大文件流式读取、大数据处理、无限序列生成、协程与异步编程、内存敏感型业务场景。

---
