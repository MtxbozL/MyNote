
Python 类中定义的方法分为三类：**实例方法、类方法、静态方法**，三者在绑定对象、参数规则、访问权限、适用场景上存在本质差异。

### 4.4.1 三类方法核心对比

|特性|实例方法|类方法|静态方法|
|---|---|---|---|
|装饰器|无需装饰器|`@classmethod`|`@staticmethod`|
|强制第一个参数|`self`（实例对象引用）|`cls`（类对象引用）|无强制参数|
|绑定对象|绑定实例对象|绑定类对象|不绑定任何对象，本质是普通函数|
|访问权限|可访问实例属性、类属性、其他实例方法、类方法、静态方法|可访问类属性、其他类方法、静态方法；无法直接访问实例属性|无法直接访问实例属性、类属性；仅能通过传参访问类或实例|
|调用方式|仅能通过实例调用|可通过类调用，也可通过实例调用|可通过类调用，也可通过实例调用|

### 4.4.2 类方法详解

类方法是绑定在**类对象**上的方法，通过`@classmethod`装饰器声明，第一个强制参数为`cls`，代表当前类本身，由 Python 解释器自动传递，无需手动传参。

核心适用场景：

1. **工厂方法**：用于创建类的实例，实现多样化的实例化方式，替代重载的构造方法；
2. **修改类属性**：用于批量修改、操作类属性，保证所有实例同步生效；
3. **类级别的工具方法**：方法逻辑与类相关，无需访问实例数据，仅需访问类属性。

示例：工厂方法实现

```python
class Student:
    def __init__(self, name, age, student_id):
        self.name = name
        self.age = age
        self.student_id = student_id
    
    @classmethod
    def from_string(cls, info_str):
        """工厂方法：从字符串解析信息，创建学生实例"""
        name, age, student_id = info_str.split(",")
        return cls(name, int(age), student_id)
    
    @classmethod
    def set_school_name(cls, new_name):
        """修改类属性"""
        cls.school_name = new_name

# 通过类方法创建实例
stu = Student.from_string("张三,18,2024001")
print(stu.name, stu.age)  # 输出：张三 18

# 调用类方法修改类属性
Student.set_school_name("XX大学")
print(Student.school_name)  # 输出：XX大学
```

### 4.4.3 静态方法详解

静态方法是放在类的命名空间中的**普通函数**，通过`@staticmethod`装饰器声明，无强制参数，不绑定类对象，也不绑定实例对象，与类的唯一关联是存储在类的命名空间中，避免全局函数的命名污染。

核心适用场景：

1. **工具类方法**：方法逻辑与类相关，但无需访问类属性、实例属性，仅作为工具函数使用；
2. **无状态方法**：方法执行不依赖类或实例的任何状态，仅处理传入的参数，返回结果。

示例：

```python
class StudentUtils:
    @staticmethod
    def is_valid_student_id(student_id):
        """静态工具方法：校验学号是否合法，无需访问类/实例属性"""
        return isinstance(student_id, str) and len(student_id) == 7 and student_id.isdigit()

# 直接通过类调用静态方法，无需创建实例
print(StudentUtils.is_valid_student_id("2024001"))  # 输出：True
print(StudentUtils.is_valid_student_id("12345"))     # 输出：False
```

---
