
推导式是 Python 特有的、极具 Pythonic 风格的语法糖，用于通过简洁的表达式快速创建容器对象，底层为 C 实现的循环逻辑，执行效率显著高于纯 Python 层的 for 循环，同时大幅提升代码简洁度与可读性。Python 支持列表、字典、集合三种推导式，无元组推导式（对应为生成器表达式）。

### 2.5.1 列表推导式

列表推导式用于快速创建列表，核心语法格式：

```python
[表达式 for 变量 in 可迭代对象 if 条件表达式]
```

执行逻辑：遍历可迭代对象，对每个满足条件的变量执行表达式，将表达式结果依次存入新列表。

核心用法：

1. **基础无过滤**：

    ```python
    # 生成0-9的平方列表
    squares = [i**2 for i in range(10)]
    # 输出：[0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
    ```
    
2. **单条件过滤**：

    ```python
    # 生成0-99中的偶数列表
    evens = [i for i in range(100) if i % 2 == 0]
    ```
    
3. **多条件过滤**：

    ```python
    # 生成0-99中大于50的3的倍数
    nums = [i for i in range(100) if i > 50 if i % 3 == 0]
    ```
    
4. **带 if-else 的三元表达式**：
    
    注意：条件判断作为表达式时，需放在 for 循环之前，与过滤条件位置不同

    ```python
    # 偶数保留，奇数乘2
    nums = [i if i % 2 == 0 else i * 2 for i in range(10)]
    ```
    
5. **嵌套循环推导式**：
    
    循环顺序与普通 for 循环一致，外层循环在前，内层循环在后

    ```python
    # 展开二维列表
    matrix = [[1,2,3], [4,5,6], [7,8,9]]
    flat = [num for row in matrix for num in row]
    # 输出：[1,2,3,4,5,6,7,8,9]
    ```
    

### 2.5.2 字典推导式

字典推导式用于快速创建字典，核心语法格式：

```python
{键表达式: 值表达式 for 变量 in 可迭代对象 if 条件表达式}
```

核心用法：

1. **基础创建**：

    ```python
    # 生成数字到平方的映射
    square_map = {i: i**2 for i in range(5)}
    # 输出：{0: 0, 1: 1, 2: 4, 3: 9, 4: 16}
    ```
    
2. **字典键值反转**：
 
    ```python
    # 仅适用于值可哈希且唯一的字典
    original = {"a": 1, "b": 2, "c": 3}
    reversed_map = {v: k for k, v in original.items()}
    # 输出：{1: "a", 2: "b", 3: "c"}
    ```
    
3. **条件过滤**：

    ```python
    # 过滤字典中值大于2的键值对
    filtered = {k: v for k, v in original.items() if v > 2}
    # 输出：{"c": 3}
    ```
    
4. **双列表生成字典**：

    ```python
    keys = ["name", "age", "gender"]
    values = ["Python", 3.12, "backend"]
    result = {k: v for k, v in zip(keys, values)}
    ```

### 2.5.3 集合推导式

集合推导式用于快速创建集合，自动去重，核心语法格式：

```python
{表达式 for 变量 in 可迭代对象 if 条件表达式}
```

核心用法：

1. **去重与转换**：

    ```python
    # 提取列表中的偶数并去重
    nums = [1,2,2,3,4,4,5,6]
    even_set = {i for i in nums if i % 2 == 0}
    # 输出：{2,4,6}
    ```
    
2. **数据预处理**：

    ```python
    # 提取字符串中的唯一单词，统一转为小写
    words = ["Python", "Java", "PYTHON", "C++", "Java"]
    unique_words = {word.lower() for word in words}
    # 输出：{"python", "java", "c++"}
    ```

### 2.5.4 使用规范

遵循 PEP8 代码规范，推导式需保持简洁可读：

1. 避免超过 2 层嵌套循环，复杂逻辑应使用普通 for 循环实现；
2. 避免过长的表达式，复杂计算应提前封装为函数；
3. 过滤条件过多时，应拆分逻辑，避免推导式可读性下降。

---
