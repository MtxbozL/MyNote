### 6.4.1 raise 关键字：主动抛出异常

`raise`关键字用于主动抛出指定的异常，中断当前代码执行，进入异常处理流程，是主动触发异常的唯一方式，支持 3 种核心用法：

1. **抛出异常类**：`raise 异常类`，自动创建异常类的无参实例并抛出，示例：

    ```python
    def divide(a, b):
        if b == 0:
            raise ZeroDivisionError  # 抛出除零异常
        return a / b
    ```
    
2. **抛出异常实例**：`raise 异常类(异常信息)`，创建带自定义错误信息的异常实例并抛出，是最常用的用法，示例：

    ```python
    def divide(a, b):
        if not isinstance(a, (int, float)) or not isinstance(b, (int, float)):
            raise TypeError("除数和被除数必须为数字类型")
        if b == 0:
            raise ZeroDivisionError("除数不能为0")
        return a / b
    ```
    
3. **重新抛出当前异常**：`raise`，无任何参数，在 except 块中使用，重新抛出当前捕获的异常，用于异常记录后向上传递，示例：

    ```python
    try:
        divide(10, 0)
    except ZeroDivisionError as e:
        print(f"捕获到异常：{e}，记录日志")
        raise  # 重新抛出异常，向上传递
    ```

### 6.4.2 自定义异常

Python 允许通过继承`Exception`类创建自定义异常，用于区分业务场景中的不同错误类型，实现精细化的异常处理，是大型项目异常体系构建的核心基础。

#### 1. 自定义异常的规范

- 自定义异常**必须直接或间接继承`Exception`类**，禁止继承`BaseException`类；
- 异常类名以`Error`为后缀，遵循大驼峰命名法，明确标识异常类型，如`UserNotFoundError`、`OrderPayError`；
- 自定义异常应按业务领域分层，如基础异常类→业务模块异常类→具体错误异常类，形成清晰的异常体系；
- 可重写`__init__`方法，携带自定义的错误信息、错误码、业务数据等，提升异常的信息携带能力。

#### 2. 自定义异常的实现示例

基础业务异常体系实现：

```python
# 项目基础异常类，所有业务异常的父类
class BusinessError(Exception):
    """项目业务异常基类"""
    def __init__(self, error_code: int, error_msg: str, data: dict = None):
        self.error_code = error_code  # 业务错误码
        self.error_msg = error_msg    # 错误信息
        self.data = data              # 异常关联的业务数据
        super().__init__(f"[{error_code}] {error_msg}")

# 用户模块异常类，继承基础业务异常
class UserError(BusinessError):
    """用户模块异常基类"""
    pass

# 具体的用户异常类
class UserNotFoundError(UserError):
    """用户不存在异常"""
    def __init__(self, user_id: str):
        super().__init__(
            error_code=404001,
            error_msg=f"用户ID[{user_id}]不存在",
            data={"user_id": user_id}
        )

class UserPasswordError(UserError):
    """用户密码错误异常"""
    def __init__(self, user_id: str):
        super().__init__(
            error_code=401001,
            error_msg=f"用户ID[{user_id}]密码错误",
            data={"user_id": user_id}
        )
```

#### 3. 自定义异常的使用

自定义异常可与标准异常一样，通过`raise`抛出，通过`except`精准捕获，实现精细化的异常处理：

```python
def get_user_info(user_id: str):
    # 模拟数据库查询
    user = None
    if not user:
        raise UserNotFoundError(user_id)
    return user

# 业务调用与异常处理
try:
    get_user_info("123456")
except UserNotFoundError as e:
    print(f"用户不存在：{e.error_msg}，错误码：{e.error_code}")
    # 执行用户不存在的兜底逻辑
except UserPasswordError as e:
    print(f"密码错误：{e.error_msg}")
    # 执行密码错误的处理逻辑
except BusinessError as e:
    print(f"业务异常：{e}")
    # 通用业务异常兜底
except Exception as e:
    print(f"系统异常：{e}")
    # 系统异常兜底
```

### 6.4.3 异常链

Python 3 支持异常链，可在捕获一个异常后，抛出另一个异常时保留原始异常的栈信息，通过`raise 新异常 from 原始异常`语法实现，用于异常的封装与转换，同时保留完整的异常上下文，便于问题排查。

示例：异常链实现

```python
try:
    # 底层数据库异常
    {}["user_id"]
except KeyError as original_e:
    # 封装为业务异常抛出，保留原始异常
    raise UserNotFoundError("123456") from original_e
```

执行该代码时，异常栈会同时显示原始的`KeyError`和封装后的`UserNotFoundError`，明确异常的完整触发链路。

若要抑制原始异常，使用`raise 新异常 from None`，仅抛出新异常，隐藏原始异常栈信息。

---
