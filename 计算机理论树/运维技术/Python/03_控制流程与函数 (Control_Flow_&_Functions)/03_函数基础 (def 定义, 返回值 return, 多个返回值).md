
函数是 Python 中**代码封装、复用、抽象的核心单元**，本质是实现特定功能的可重复调用的代码块，通过函数可将复杂程序拆分为多个独立、低耦合的功能模块，是工程化开发的核心基础。

### 3.3.1 函数的定义与调用

Python 通过`def`关键字定义函数，完整语法结构如下：

```python
def 函数名(参数列表):
    """函数文档字符串（docstring）"""
    # 函数体代码块
    函数体语句
    return 返回值
```

核心规则：

1. **函数名规范**：遵循 PEP8 规范，使用小写字母，多个单词用下划线分隔（snake_case），禁止使用关键字、内置函数名作为函数名，避免命名空间污染；
2. **参数列表**：可选，用于接收调用方传入的数据，支持多种参数类型（3.4 节详细讲解），无参数时保留空括号；
3. **文档字符串（docstring）**：可选，用于描述函数的功能、参数、返回值、异常等信息，是 Python 官方推荐的函数注释方式，可通过`函数名.__doc__`属性访问，也可通过`help(函数名)`查看，工业界通用 Google、NumPy、reStructuredText 三种规范；
4. 函数必须先定义（def 执行），后调用，否则会抛出`NameError`；
5. 函数调用：通过`函数名(实参列表)`的方式调用，实参与形参按参数规则匹配，调用时触发函数体代码执行。

### 3.3.2 return 语句与返回值

return 语句用于**终止函数的执行**，并将指定结果返回给调用方，核心规则：

1. 函数执行到 return 语句时立即终止，return 之后的代码不会执行；
2. 无 return 语句、return 后无值的函数，默认返回`None`；
3. 支持返回多个值，本质是将多个值打包为**元组**返回，调用方可通过元组解包接收，与第二章元组解包机制完全联动，示例：

    ```python
    def calculate(a, b):
        sum_ab = a + b
        product_ab = a * b
        return sum_ab, product_ab  # 等价于 return (sum_ab, product_ab)
    
    # 解包接收返回值
    sum_result, product_result = calculate(3, 4)
    print(sum_result, product_result)  # 输出：7 12
    
    # 接收完整元组
    result_tuple = calculate(3, 4)
    print(result_tuple)  # 输出：(7, 12)
    ```
    
4. 支持返回任意类型的 Python 对象，包括列表、字典、函数、类等，为高阶函数、闭包等特性提供支持。

---
