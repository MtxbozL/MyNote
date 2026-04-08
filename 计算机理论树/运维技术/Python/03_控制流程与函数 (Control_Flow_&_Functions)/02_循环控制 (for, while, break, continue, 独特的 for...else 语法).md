
Python 提供`while`与`for`两种循环结构，用于实现重复执行的代码逻辑，同时提供`break`、`continue`实现循环流程的精细控制，以及 Python 特有的`for...else`/`while...else`语法。

### 3.2.1 while 循环

while 循环适用于**循环次数不确定**的场景，基于条件表达式的布尔值控制循环执行，语法结构如下：

```python
while 条件表达式:
    # 条件为True时循环执行的代码块
    循环体语句
else:
    # 循环正常结束（未被break终止）时执行的代码块
    语句
```

核心规则：

1. 条件表达式在每次循环开始前判断，为 True 则执行循环体，为 False 则终止循环；
2. 无限循环：条件表达式恒为`True`时形成无限循环，必须在循环体内通过`break`设置退出条件，示例：

    ```python
    while True:
        user_input = input("请输入q退出：")
        if user_input == "q":
            break
    ```
    
3. 循环变量必须在循环体内更新，否则会导致条件恒为 True，形成死循环。

### 3.2.2 for 循环

for 循环是 Python 中最常用的循环结构，适用于**遍历可迭代对象**的场景，循环次数由可迭代对象的元素个数决定，核心本质是基于**迭代器协议**实现的遍历（第五章详细讲解），语法结构如下：

```python
for 临时变量 in 可迭代对象:
    # 遍历每个元素时执行的代码块
    循环体语句
else:
    # 循环正常结束（未被break终止）时执行的代码块
    语句
```

核心规则与用法：

1. 可迭代对象包括：字符串、列表、元组、字典、集合、range 对象、生成器等所有实现了迭代器协议的对象；
2. `range()`函数：用于生成整数序列的可迭代对象，是 for 循环计数场景的核心工具，语法为`range(start, stop, step)`，参数规则与第二章切片完全一致：
    
    - `range(n)`：生成 0 到 n-1 的整数序列，步长 1；
    - `range(m, n)`：生成 m 到 n-1 的整数序列，步长 1；
    - `range(m, n, s)`：生成 m 到 n-1 的整数序列，步长 s，s 为负数时反向生成；
    
3. 字典遍历：`for k in dict`遍历键，`for k, v in dict.items()`遍历键值对（官方推荐方式）；
4. 作用域特性：Python 的代码块（if/for/while）不会创建新的作用域，循环结束后，临时变量仍会保留最后一次遍历的值，可在循环外部访问。

### 3.2.3 break 与 continue

二者均为循环流程控制关键字，仅作用于当前所在的内层循环，嵌套循环中不会影响外层循环：

1. **break**：立即**终止整个当前循环**，跳出循环体，循环的 else 块不会执行；
2. **continue**：立即**跳过当前循环迭代的剩余代码**，直接进入下一次循环的条件判断，不会终止整个循环。

示例对比：

```python
# break示例：遇到3立即终止循环
for i in range(5):
    if i == 3:
        break
    print(i)  # 输出：0 1 2

# continue示例：遇到3跳过当前迭代，继续循环
for i in range(5):
    if i == 3:
        continue
    print(i)  # 输出：0 1 2 4
```

### 3.2.4 循环 - else 语法（Python 特有）

这是 Python 区别于其他编程语言的核心语法特性，`else`块与循环绑定，而非与 if 绑定。**执行规则：仅当循环正常执行完毕（所有迭代完成，未被 break 强制终止）时，else 块才会执行；若循环被 break 终止，else 块完全不执行**。

核心适用场景：遍历查找元素，找到则 break 终止循环，未找到则在 else 块执行兜底逻辑，无需额外的标记变量，示例：

```python
nums = [1, 2, 3, 4, 5]
target = 6

for num in nums:
    if num == target:
        print(f"找到目标元素{target}")
        break
else:
    # 仅当循环遍历完所有元素、未触发break时执行
    print(f"未找到目标元素{target}")
```

注意：while 循环的 else 块遵循完全相同的规则。

---
