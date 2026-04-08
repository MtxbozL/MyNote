
自动化测试是 Python 项目质量保障的核心手段，通过自动化用例实现代码逻辑的持续验证，保证代码修改不会引入新的 bug，是敏捷开发、CI/CD 流程的核心基础。Python 官方提供`unittest`单元测试框架，工业界主流使用更简洁、更强大的`pytest`测试框架，同时通过`mock`实现外部依赖的模拟，完成隔离测试。

### 8.3.1 测试核心概念

- **单元测试**：对软件中最小的可测试单元（函数、方法、类）进行隔离测试，验证其逻辑正确性，核心目标是覆盖代码的所有分支、边界条件；
- **集成测试**：对多个模块、多个单元的协同工作进行测试，验证模块间的交互逻辑是否符合预期；
- **测试用例**：针对特定场景设计的测试代码，包含输入、预期输出、执行步骤，用于验证代码的特定逻辑；
- **断言**：判断代码执行结果是否符合预期的表达式，断言失败则测试用例不通过；
- **测试隔离**：每个测试用例之间相互独立，互不影响，一个用例的失败不会导致其他用例执行异常；
- **Mock 测试**：模拟代码中的外部依赖（如数据库、API 请求、第三方服务），让测试仅关注核心逻辑，不受外部依赖的可用性影响。

### 8.3.2 unittest 官方单元测试框架

`unittest`（又称 PyUnit）是 Python 内置的官方单元测试框架，基于 xUnit 设计模式，无需额外安装，兼容性极强，适用于简单项目、官方标准库的测试开发。

#### 1. 核心结构与用法

- 测试用例通过继承`unittest.TestCase`类实现；
- 测试方法必须以`test_`为前缀，否则不会被执行；
- 提供`setUp()`/`tearDown()`方法，在每个测试方法执行前后执行，用于测试环境的初始化与清理；
- 提供`setUpClass()`/`tearDownClass()`类方法，在整个测试类的所有用例执行前后执行，仅执行一次；
- 提供丰富的断言方法，用于验证执行结果。

示例：完整的 unittest 测试用例

```python
import unittest
from my_project.calc import add, divide, is_prime

# 测试类必须继承unittest.TestCase
class TestCalc(unittest.TestCase):
    # 整个测试类执行前执行一次
    @classmethod
    def setUpClass(cls):
        print("测试类初始化，准备全局测试数据")
        cls.test_data = [1, 2, 3, 4, 5]

    # 每个测试方法执行前执行
    def setUp(self):
        print("单个用例初始化")
        self.a = 10
        self.b = 2

    # 每个测试方法执行后执行
    def tearDown(self):
        print("单个用例清理")

    # 整个测试类执行完成后执行一次
    @classmethod
    def tearDownClass(cls):
        print("测试类清理")

    # 测试方法，必须以test_开头
    def test_add(self):
        # 断言：验证结果是否符合预期
        self.assertEqual(add(self.a, self.b), 12)
        self.assertEqual(add(-1, 1), 0)
        self.assertEqual(add(0, 0), 0)

    def test_divide(self):
        self.assertEqual(divide(self.a, self.b), 5)
        # 断言是否抛出指定异常
        with self.assertRaises(ZeroDivisionError):
            divide(self.a, 0)

    def test_is_prime(self):
        self.assertTrue(is_prime(2))
        self.assertTrue(is_prime(7))
        self.assertFalse(is_prime(1))
        self.assertFalse(is_prime(10))

# 执行测试
if __name__ == '__main__':
    unittest.main()
```

#### 2. 核心断言方法

|断言方法|功能描述|
|---|---|
|`assertEqual(a, b)`|断言 a == b|
|`assertNotEqual(a, b)`|断言 a != b|
|`assertTrue(x)`|断言 x 为 True|
|`assertFalse(x)`|断言 x 为 False|
|`assertIs(a, b)`|断言 a is b|
|`assertIsNone(x)`|断言 x is None|
|`assertIn(a, b)`|断言 a in b|
|`assertRaises(exc, func, *args)`|断言 func 执行时抛出 exc 异常|
|`assertAlmostEqual(a, b)`|断言 a 和 b 的浮点值近似相等，解决浮点数精度问题|

### 8.3.3 pytest 现代测试框架【工业界首选】

`pytest`是 Python 最主流的现代测试框架，兼容`unittest`测试用例，同时提供更简洁的语法、更强大的功能、更丰富的插件生态，大幅降低测试代码的编写成本，是企业级项目的首选测试框架。

#### 1. 核心优势与安装

- 无需继承基类，普通函数即可编写测试用例；
- 兼容 unittest 测试用例，无缝迁移存量代码；
- 强大的 fixture 机制，实现测试数据、测试环境的灵活复用；
- 丰富的参数化测试能力，简化多场景用例编写；
- 详细的失败信息输出，快速定位问题；
- 超过 1000 个第三方插件，支持 allure 报告、并发测试、覆盖率统计等全场景需求。

安装命令：`poetry add pytest --group dev` 或 `pip install pytest`

#### 2. 基础测试用例编写

pytest 的测试用例编写规则极简：

- 测试文件必须以`test_`开头或`_test.py`结尾；
- 测试函数必须以`test_`开头；
- 测试类（可选）必须以`Test`开头，且不能有`__init__`方法；
- 直接使用 Python 原生`assert`关键字进行断言，无需记忆复杂的断言方法。

示例：pytest 基础测试用例

```python
# test_calc.py
from my_project.calc import add, divide, is_prime

# 普通函数作为测试用例
def test_add():
    assert add(10, 2) == 12
    assert add(-1, 1) == 0
    assert add(0, 0) == 0

def test_divide():
    assert divide(10, 2) == 5
    # 断言异常
    with pytest.raises(ZeroDivisionError):
        divide(10, 0)

# 测试类
class TestPrime:
    def test_prime_positive(self):
        assert is_prime(2) is True
        assert is_prime(7) is True

    def test_prime_negative(self):
        assert is_prime(1) is False
        assert is_prime(10) is False
```

执行测试：在项目根目录执行`pytest`命令，pytest 会自动发现所有符合规则的测试文件与用例，执行并输出测试结果。

常用执行参数：

```bash
# 执行所有测试用例
pytest

# 执行指定文件的测试用例
pytest tests/test_calc.py

# 执行指定测试函数
pytest tests/test_calc.py::test_add

# 执行指定测试类的用例
pytest tests/test_calc.py::TestPrime

# 输出详细的执行信息
pytest -v

# 输出print的内容
pytest -s

# 仅执行上次失败的用例
pytest --lf

# 执行匹配关键字的用例
pytest -k "add or prime"
```

#### 3. fixture 机制【核心高频考点】

fixture 是 pytest 的核心灵魂，用于实现测试数据、测试环境的初始化、复用与清理，替代了 unittest 的`setUp()`/`tearDown()`，灵活性、复用性远超传统方案。

fixture 的核心特性：

- 可通过函数名直接注入到测试用例中，无需继承、无需手动调用；
- 支持灵活的作用域控制，从单个用例到整个测试会话均可覆盖；
- 支持依赖注入，一个 fixture 可以依赖其他 fixture；
- 支持 setup 和 teardown 双向逻辑，通过 yield 关键字分离初始化与清理代码；
- 可通过`conftest.py`实现全局共享，无需导入即可在整个项目中使用。

##### （1）基础 fixture 定义与使用

```python
import pytest

# 定义fixture
@pytest.fixture
def test_user():
    # yield之前的代码：setup初始化，测试用例执行前运行
    user = {"id": 1, "name": "张三", "age": 18}
    print("初始化测试用户数据")
    yield user  # 返回测试数据，注入到测试用例中
    # yield之后的代码：teardown清理，测试用例执行后运行
    print("清理测试用户数据")

# 测试用例通过参数名注入fixture
def test_user_age(test_user):
    assert test_user["age"] == 18

def test_user_id(test_user):
    assert test_user["id"] == 1
```

##### （2）fixture 作用域控制

通过`scope`参数控制 fixture 的生命周期，可选值如下：

|作用域|描述|
|---|---|
|`function`|默认值，每个测试函数执行一次，每个用例独立初始化|
|`class`|每个测试类执行一次，类内所有用例共享同一个 fixture|
|`module`|每个.py 测试文件执行一次，文件内所有用例共享|
|`package`|每个包执行一次，包内所有测试文件共享|
|`session`|整个测试会话执行一次，所有测试用例共享|

示例：数据库连接 fixture，整个测试会话仅创建一次

```python
import pytest
import pymysql

@pytest.fixture(scope="session")
def db_connection():
    # 整个测试会话开始时，创建数据库连接
    conn = pymysql.connect(host="localhost", user="test", password="123456", database="test_db")
    yield conn
    # 整个测试会话结束时，关闭连接
    conn.close()
```

##### （3）conftest.py 全局共享 fixture

`conftest.py`是 pytest 的全局配置文件，在该文件中定义的 fixture，可在整个项目的测试用例中直接使用，无需手动导入。pytest 会自动向上递归查找`conftest.py`文件，加载其中的 fixture。

工程化最佳实践：

- 在项目的`tests`目录下创建`conftest.py`，存放项目全局通用的 fixture；
- 子模块的测试目录下可创建独立的`conftest.py`，存放该模块专用的 fixture；
- 禁止在测试文件中手动导入`conftest.py`，pytest 会自动加载。

#### 4. 参数化测试

pytest 通过`@pytest.mark.parametrize`装饰器实现参数化测试，可将多组测试数据与预期结果传入同一个测试函数，自动生成多个测试用例，大幅减少重复代码。

示例：多场景参数化测试

```python
import pytest

# 单参数参数化
@pytest.mark.parametrize("num, expected", [
    (2, True),
    (7, True),
    (1, False),
    (10, False),
    (0, False)
])
def test_is_prime(num, expected):
    assert is_prime(num) == expected

# 多参数组合参数化，生成笛卡尔积用例
@pytest.mark.parametrize("a", [1, 2, 3])
@pytest.mark.parametrize("b", [4, 5, 6])
def test_add_multi(a, b):
    assert add(a, b) == a + b
```

#### 5. 常用标记与插件

- **用例标记**：通过`@pytest.mark`为用例添加自定义标记，实现用例的分组执行，示例：

    ```python
    # 标记慢用例
    @pytest.mark.slow
    def test_large_calc():
        assert is_prime(1000003) is True
    ```
    
    执行时仅运行标记的用例：`pytest -m slow`，跳过标记的用例：`pytest -m "not slow"`。
- **常用插件**：

| 插件名             | 功能描述                           |
| --------------- | ------------------------------ |
| `pytest-cov`    | 代码覆盖率统计，生成覆盖率报告，验证测试用例的代码覆盖程度  |
| `pytest-xdist`  | 分布式并发执行测试用例，大幅缩短大型项目的测试执行时间    |
| `pytest-html`   | 生成 HTML 格式的测试报告                |
| `pytest-mock`   | 集成 unittest.mock，简化 mock 测试的编写 |
| `allure-pytest` | 生成 allure 美观的测试报告，企业级测试报告首选    |

### 8.3.4 Mock 测试

Mock 测试的核心是**模拟代码中的外部依赖**，隔离核心测试逻辑，避免外部依赖（数据库、第三方 API、网络服务）的不可用、不稳定对测试结果的影响，同时可以验证外部依赖的调用是否符合预期。Python 通过内置的`unittest.mock`模块实现 Mock 测试，pytest 可通过`pytest-mock`插件简化使用。

#### 1. 核心概念与常用类

- `Mock`/`MagicMock`：模拟对象，可替代任何 Python 对象，记录调用信息，自定义返回值、抛出异常；
- `patch`：装饰器 / 上下文管理器，用于临时替换指定的对象，测试结束后自动恢复，是 Mock 测试的核心工具。

#### 2. 核心用法示例

场景：测试一个调用第三方 API 的函数，模拟 API 请求，避免真实网络请求。

待测试代码：`user_api.py`

```python
import requests

def get_user_info(user_id: int):
    """调用第三方API获取用户信息"""
    response = requests.get(f"https://api.example.com/users/{user_id}")
    if response.status_code == 200:
        return response.json()
    raise ValueError("用户不存在")
```

测试代码：使用 patch 模拟 requests.get

```python
import pytest
from unittest.mock import patch, MagicMock
from my_project.user_api import get_user_info

def test_get_user_info_success():
    # 定义模拟的返回数据
    mock_response = MagicMock()
    mock_response.status_code = 200
    mock_response.json.return_value = {"id": 1, "name": "张三"}

    # 使用patch替换requests.get，模拟API请求
    with patch("my_project.user_api.requests.get", return_value=mock_response) as mock_get:
        # 调用待测试函数
        result = get_user_info(1)
        # 断言结果符合预期
        assert result["id"] == 1
        assert result["name"] == "张三"
        # 断言模拟的函数被正确调用，参数符合预期
        mock_get.assert_called_once_with("https://api.example.com/users/1")

def test_get_user_info_failed():
    mock_response = MagicMock()
    mock_response.status_code = 404

    with patch("my_project.user_api.requests.get", return_value=mock_response):
        # 断言抛出指定异常
        with pytest.raises(ValueError):
            get_user_info(999)
```

#### 3. 核心高级用法

- **模拟异常**：通过`side_effect`设置模拟函数调用时抛出的异常，示例：

    ```python
    with patch("requests.get", side_effect=ConnectionError("网络异常")):
        with pytest.raises(ConnectionError):
            get_user_info(1)
    ```
    
- **多次调用返回不同值**：`side_effect`传入列表，每次调用返回列表中的下一个值，示例：

    ```python
    mock_func = MagicMock(side_effect=[1, 2, 3])
    print(mock_func()) # 1
    print(mock_func()) # 2
    print(mock_func()) # 3
    ```

- **调用验证**：通过`assert_called()`、`assert_called_once()`、`assert_called_with()`、`assert_any_call()`等方法，验证模拟函数的调用次数、调用参数是否符合预期。

### 8.3.5 测试最佳实践

1. **测试用例隔离**：每个测试用例必须相互独立，不能依赖其他用例的执行结果，不能共享可变的测试数据；
2. **单一职责**：每个测试用例仅验证一个逻辑点，避免一个用例包含多个断言、多个测试场景；
3. **覆盖边界条件**：测试用例必须覆盖正常场景、边界条件、异常场景、非法输入，保证代码的鲁棒性；
4. **测试数据与逻辑分离**：测试数据通过 fixture、参数化管理，避免硬编码在测试逻辑中；
5. **禁止在测试中使用真实外部依赖**：所有外部依赖必须通过 Mock 模拟，保证测试的稳定性、可重复性；
6. **测试驱动开发（TDD）**：优先编写测试用例，再编写业务代码，通过测试用例驱动代码设计，保证代码的可测试性；
7. **持续集成**：将自动化测试集成到 CI/CD 流程中，代码提交时自动执行测试，不通过测试禁止合并代码。

---
