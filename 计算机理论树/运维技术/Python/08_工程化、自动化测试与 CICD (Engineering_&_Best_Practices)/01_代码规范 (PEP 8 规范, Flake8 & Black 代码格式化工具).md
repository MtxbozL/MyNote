## 本章概述

本章是 Python 从单文件脚本开发到企业级工程化落地的核心跃迁章节，覆盖 Python 项目全生命周期的工程化规范与最佳实践。核心围绕**代码规范性、依赖管理、质量保障、可观测性、分发发布**五大工程化核心模块展开，彻底解决脚本开发中的代码混乱、依赖地狱、质量不可控、调试困难、分发难等核心痛点。所有内容严格遵循 Python 官方 PEP 规范与工业界企业级项目标准，深度覆盖后端开发、开源项目维护、团队协作场景的核心要求，同时包含 Python 面试中工程化方向的高频考点。

---

代码规范是团队协作、项目可维护性的基础，Python 官方通过**PEP 8**规范定义了 Python 代码的官方编码风格指南，同时配套自动化工具链实现规范的强制落地，避免人工校验的疏漏与分歧。

### 8.1.1 PEP 8 核心规范

PEP 8 是 Python 社区唯一官方认可的编码规范，所有 Python 代码均应遵循该规范，核心内容分为以下六大核心模块，其中强制规范必须严格遵守，推荐规范应在所有场景落地。

#### 1. 缩进规范【强制】

- 代码缩进必须使用**4 个空格**，禁止使用 Tab 键，禁止 Tab 与空格混合使用；
- 行尾括号换行时，换行后的代码应与左括号对齐，或使用额外 4 个空格的悬挂缩进；
- 禁止使用反斜杠`\`进行行连接，优先使用括号包裹多行表达式。

正确示例：

```python
# 括号对齐换行
def long_function_name(
        param_one, param_two, param_three,
        param_four, param_five):
    print(param_one)

# 括号包裹多行表达式
total = (var_one + var_two + var_three
         - var_four * var_five)
```

#### 2. 行长度与空行规范【强制】

- 代码行最大长度限制为**79 个字符**，文档字符串、注释行最大长度为 72 个字符；
- 顶级函数、类定义之间空 2 行；
- 类内部的方法定义之间空 1 行；
- 函数内部的逻辑块之间用 1 个空行分隔，禁止连续多个空行；
- 禁止在行尾添加多余的空格。

#### 3. 导入规范【强制】

- 导入语句必须放在文件顶部，位于模块注释、文档字符串之后，全局变量、常量定义之前；
- 导入顺序必须遵循**标准库→第三方库→本地项目模块**的顺序，三组之间用 1 个空行分隔；
- 每行仅导入一个模块，禁止一行导入多个模块；
- 禁止使用通配符导入`from module import *`，会严重污染命名空间，仅允许在`__init__.py`中配合`__all__`显式控制时使用；
- 优先使用绝对导入，禁止使用复杂的相对导入；
- 禁止导入未使用的模块。

正确示例：

```python
# 标准库导入
import os
import sys
from datetime import datetime

# 第三方库导入
import requests
import pandas as pd

# 本地项目导入
from utils.logger import get_logger
from config.settings import BASE_DIR
```

#### 4. 命名规范【强制】

Python 命名规范的核心原则是：**命名风格直接体现变量 / 对象的类型与作用域**，不同类型的对象必须使用对应的命名风格，禁止混用。

|对象类型|命名规范|示例|核心规则|
|---|---|---|---|
|类、异常名|大驼峰命名法（PascalCase）|`UserService`, `FileNotFoundError`|每个单词首字母大写，无下划线分隔，异常名必须以`Error`为后缀|
|函数、变量、方法名|蛇形命名法（snake_case）|`get_user_info`, `max_count`|全小写字母，多个单词用下划线分隔|
|模块、包名|短蛇形命名法|`user_utils`, `api_gateway`|全小写字母，尽量短，避免下划线（除非能显著提升可读性）|
|全局常量|全大写下划线分隔|`MAX_RETRY_TIMES`, `DEFAULT_TIMEOUT`|全大写，多个单词用下划线分隔，仅模块顶层定义|
|受保护成员|单下划线前缀蛇形命名|`_private_method`, `_internal_var`|仅模块内部 / 类内部、子类可访问，外部禁止直接访问|
|私有成员|双下划线前缀蛇形命名|`__secret_key`, `__core_calc`|仅类内部可访问，解释器会执行名称改写，禁止外部访问|
|实例方法第一个参数|固定为`self`|-|禁止修改为其他名称|
|类方法第一个参数|固定为`cls`|-|禁止修改为其他名称|

#### 5. 表达式与语句规范【强制】

- 禁止在单行中编写多条语句，禁止`if/for/while`单行简写；
- 二元运算符两侧必须加 1 个空格，逗号后必须加 1 个空格，括号内侧禁止加空格；
- 函数参数的默认值等号两侧禁止加空格；
- 禁止使用`==`/`!=`与`None`比较，必须使用`is`/`is not`；
- 布尔值判断直接使用`if is_valid:`，禁止使用`if is_valid == True:`；
- 异常捕获必须精准捕获异常类型，禁止使用空`except:`捕获所有异常。

正确示例：

```python
# 正确的运算符空格
x = a + b * c
# 正确的函数默认参数
def func(a, b=0, c=None):
    pass
# 正确的None判断
if obj is None:
    pass
# 正确的布尔判断
if not user_list:
    pass
```

#### 6. 注释规范【推荐】

- 注释必须清晰、准确，与代码逻辑保持一致，禁止出现与代码矛盾的注释；
- 行内注释与代码之间至少间隔 2 个空格，以`#`加 1 个空格开头；
- 块注释每一行都以`#`加 1 个空格开头，缩进与被注释的代码一致；
- 所有公共模块、函数、类、方法必须编写文档字符串（docstring），私有方法推荐编写文档字符串；
- 文档字符串推荐使用 Google 风格、NumPy 风格或 reStructuredText 风格，保持项目内风格统一。

Google 风格文档字符串示例：

```python
def calculate_user_age(birth_date: datetime) -> int:
    """根据出生日期计算用户年龄

    Args:
        birth_date: 用户的出生日期datetime对象

    Returns:
        int: 用户的周岁年龄

    Raises:
        ValueError: 当出生日期晚于当前日期时抛出
    """
    today = datetime.today()
    if birth_date > today:
        raise ValueError("出生日期不能晚于当前日期")
    age = today.year - birth_date.year
    if (today.month, today.day) < (birth_date.month, birth_date.day):
        age -= 1
    return age
```

### 8.1.2 代码规范自动化工具链

人工校验规范效率低、易出错，工业界通过自动化工具链实现规范的自动检查、自动格式化，核心工具分为**静态检查工具**与**自动格式化工具**两大类，同时通过`pre-commit`实现提交代码前的自动校验。

|工具|类型|核心功能|安装命令|
|---|---|---|---|
|`flake8`|静态检查工具|集成 PEP 8 检查、代码逻辑错误检查、复杂度检查，是 Python 静态检查的工业标准|`pip install flake8`|
|`black`|自动格式化工具|无配置、强约束的 PEP 8 兼容格式化工具，被称为 “不妥协的代码格式化器”，Python 官方推荐，彻底解决团队代码风格分歧|`pip install black`|
|`isort`|自动格式化工具|专门用于导入语句的自动排序与格式化，严格遵循 PEP 8 导入规范，可与 black 无缝配合|`pip install isort`|
|`autoflake`|自动格式化工具|自动删除未使用的导入、未使用的变量、无用的 pass 语句，清理代码冗余|`pip install autoflake`|
|`pylint`|静态检查工具|更严格的代码静态分析工具，除 PEP 8 外，还会检查代码异味、设计缺陷、可维护性，输出代码质量评分|`pip install pylint`|

#### 1. 核心工具常用命令

```python
# black格式化当前目录所有Python文件
black .

# isort格式化导入语句，兼容black配置
isort . --profile black

# autoflake清理未使用的导入和变量
autoflake --in-place --remove-all-unused-imports --remove-unused-variables ./**/*.py

# flake8静态检查当前目录
flake8 . --max-line-length=88 --extend-ignore=E203
```

#### 2. pre-commit 钩子集成【工程化最佳实践】

`pre-commit`是 Git 钩子管理工具，可在代码提交前自动执行规范检查、格式化、静态分析，不通过检查禁止提交，从源头保证代码规范的落地，是团队协作的强制规范工具。

核心使用流程：

1. 安装 pre-commit：`pip install pre-commit`
2. 项目根目录创建`.pre-commit-config.yaml`配置文件，示例：

    ```yaml
    repos:
    -   repo: https://github.com/pre-commit/pre-commit-hooks
        rev: v4.6.0
        hooks:
        -   id: trailing-whitespace # 移除行尾空格
        -   id: end-of-file-fixer # 确保文件以换行结尾
        -   id: check-yaml # 检查yaml文件语法
        -   id: check-json # 检查json文件语法
        -   id: check-merge-conflict # 检查合并冲突标记
    -   repo: https://github.com/psf/black
        rev: 24.4.2
        hooks:
        -   id: black # 自动格式化代码
    -   repo: https://github.com/PyCQA/isort
        rev: 5.13.2
        hooks:
        -   id: isort # 格式化导入语句
            args: [--profile, black]
    -   repo: https://github.com/PyCQA/flake8
        rev: 6.0.0
        hooks:
        -   id: flake8 # 静态检查
            args: [--max-line-length=88, --extend-ignore=E203]
    ```

3. 安装 Git 钩子：`pre-commit install`
4. 后续执行`git commit`时，会自动执行配置的钩子，检查不通过会终止提交，修复后才可正常提交。

---
