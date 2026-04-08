
## 9.3 Flask 轻量级微框架

Flask 是 Python 生态中最流行的**轻量级微内核 Web 框架**，遵循 “微内核 + 扩展” 的设计哲学，核心仅包含 WSGI 适配、路由分发、Jinja2 模板引擎、请求上下文管理四个核心组件，其他所有能力（ORM、表单、认证、缓存等）均通过第三方扩展实现，极致灵活、高度可定制，无强制的项目结构与架构约束，适用于小型项目、API 开发、原型验证、个性化定制场景。

### 9.3.1 核心架构与路由机制

#### 1. Flask 最小应用与核心依赖

Flask 的核心依赖仅两个：

- **Werkzeug**：WSGI 工具集，提供了路由匹配、请求 / 响应封装、HTTP 工具、WSGI 服务器适配等底层能力，是 Flask 的核心底层；
- **Jinja2**：Python 生态最主流的模板引擎，提供了服务端页面渲染能力，支持模板继承、控制语句、过滤器等完整功能。

Flask 最小应用示例：

python

运行

```
from flask import Flask

# 初始化Flask应用实例，__name__为当前模块名，用于定位资源路径
app = Flask(__name__)

# 路由装饰器，将URL路径与视图函数绑定
@app.route("/")
def index():
    """视图函数，处理请求并返回响应"""
    return "Hello Flask"

# 启动开发服务器
if __name__ == "__main__":
    app.run(debug=True)
```

启动项目后，访问`http://127.0.0.1:5000`即可看到响应内容。

#### 2. 路由机制

Flask 的路由核心是通过`@app.route()`装饰器将 URL 规则与视图函数绑定，底层基于 Werkzeug 的路由匹配器实现，支持丰富的路由规则定义。

##### （1）基础路由定义

python

运行

```
# 基础路径路由
@app.route("/about")
def about():
    return "关于我们"

# 带路径参数的路由
@app.route("/user/<int:user_id>")
def user_detail(user_id):
    # 路径参数自动转换为int类型，直接作为视图函数参数传入
    return f"用户ID：{user_id}"

# 多路径绑定同一个视图函数
@app.route("/")
@app.route("/home")
def home():
    return "首页"

# 限制请求方法，默认仅支持GET
@app.route("/login", methods=["GET", "POST"])
def login():
    return "登录页面"
```

##### （2）路径参数转换器

Flask 内置了多种参数转换器，实现路径参数的类型转换与校验：

表格

|转换器|功能描述|示例|
|---|---|---|
|`string`|默认转换器，匹配除`/`外的字符串|`/user/<string:username>`|
|`int`|匹配正整数|`/article/<int:article_id>`|
|`float`|匹配正浮点数|`/price/<float:price>`|
|`path`|匹配包含`/`的字符串，用于匹配路径|`/file/<path:file_path>`|
|`uuid`|匹配 UUID 格式的字符串|`/device/<uuid:device_id>`|

##### （3）URL 反向解析

通过`url_for()`函数，根据视图函数名生成对应的 URL 路径，避免硬编码 URL，示例：

python

运行

```
from flask import url_for

@app.route("/")
def index():
    # 生成user_detail视图的URL，参数为路径参数
    user_url = url_for("user_detail", user_id=1)
    # 生成带查询参数的URL
    login_url = url_for("login", next="/home")
    return f"用户URL：{user_url}，登录URL：{login_url}"
```

### 9.3.2 Jinja2 模板引擎

Jinja2 是 Flask 默认的模板引擎，基于文本模板生成动态 HTML 页面，完全兼容 Unicode，支持模板继承、自动 HTML 转义、沙箱执行等能力，是服务端渲染场景的核心组件。

#### 1. 模板基础用法

Flask 默认从项目根目录的`templates`文件夹中加载模板文件，通过`render_template()`函数渲染模板，示例：

视图函数：

python

运行

```
from flask import Flask, render_template

app = Flask(__name__)

@app.route("/user/<int:user_id>")
def user_detail(user_id):
    # 模板渲染所需的变量
    user = {
        "id": user_id,
        "username": "zhangsan",
        "age": 20,
        "hobbies": ["编程", "阅读", "运动"]
    }
    # 渲染templates/user_detail.html模板，传入模板变量
    return render_template("user_detail.html", user=user, title="用户详情")
```

模板文件`templates/user_detail.html`：

html

预览

```
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>{{ title }}</title>
</head>
<body>
    <h1>用户详情</h1>
    <!-- 变量渲染，双大括号{{ }} -->
    <p>用户ID：{{ user.id }}</p>
    <p>用户名：{{ user.username }}</p>
    <p>年龄：{{ user.age }}</p>

    <!-- 控制语句：if条件判断，{% %} -->
    {% if user.age >= 18 %}
        <p>成年用户</p>
    {% else %}
        <p>未成年用户</p>
    {% endif %}

    <!-- 循环语句：for循环 -->
    <h3>兴趣爱好</h3>
    <ul>
        {% for hobby in user.hobbies %}
            <li>{{ loop.index }}. {{ hobby }}</li>
        {% else %}
            <li>暂无兴趣爱好</li>
        {% endfor %}
    </ul>
</body>
</html>
```

#### 2. Jinja2 核心语法

- **变量渲染**：`{{ 变量名 }}`，支持变量的属性访问、字典取值、算术运算；
- **控制语句**：`{% 语句 %}`，支持`if/elif/else`、`for/else`、宏定义、模板继承等；
- **注释**：`{# 注释内容 #}`，不会渲染到 HTML 中；
- **过滤器**：`{{ 变量名|过滤器名 }}`，用于对变量进行格式化处理，内置高频过滤器：
    
    - `safe`：关闭 HTML 自动转义，渲染原始 HTML 内容；
    - `length`：获取变量长度；
    - `upper`/`lower`：大小写转换；
    - `format`：字符串格式化；
    - `date`：日期格式化；
    - `default`：设置默认值。
    

#### 3. 模板继承

Jinja2 的模板继承是实现模板复用的核心能力，通过`{% extends %}`继承基础模板，`{% block %}`定义可重写的区块，消除模板代码的重复，示例：

基础模板`templates/base.html`：

html

预览

```
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}Flask应用{% endblock %}</title>
    <link rel="stylesheet" href="/static/css/common.css">
    {% block head %}{% endblock %}
</head>
<body>
    <header>
        <nav>
            <a href="/">首页</a>
            <a href="/about">关于</a>
        </nav>
    </header>
    <main>
        <!-- 核心内容区块，子模板重写 -->
        {% block content %}{% endblock %}
    </main>
    <footer>
        <p>© 2024 Flask应用</p>
    </footer>
    {% block script %}{% endblock %}
</body>
</html>
```

子模板`templates/index.html`：

html

预览

```
<!-- 继承基础模板 -->
{% extends "base.html" %}

<!-- 重写title区块 -->
{% block title %}首页 - Flask应用{% endblock %}

<!-- 重写content区块 -->
{% block content %}
    <h1>首页</h1>
    <p>欢迎访问Flask应用</p>
{% endblock %}

<!-- 扩展script区块 -->
{% block script %}
    <script src="/static/js/index.js"></script>
{% endblock %}
```

#### 4. 自动转义与 XSS 防护

Jinja2 默认开启 HTML 自动转义，会将`{{ }}`中的变量内容中的`<`、`>`、`"`、`'`、`&`等特殊字符转义为 HTML 实体，彻底杜绝 XSS 跨站脚本攻击。仅当明确信任变量内容时，通过`safe`过滤器关闭转义，渲染原始 HTML。

### 9.3.3 请求上下文与 ThreadLocal 原理【面试超高频考点】

Flask 的上下文机制是框架的核心难点，也是面试高频考点，其核心解决的问题是：在多线程环境下，如何让每个请求的请求对象、应用对象在视图函数中全局可访问，且不同请求之间完全隔离，互不干扰。

#### 1. 上下文的核心分类

Flask 提供两种核心上下文，每个上下文对应两个全局变量：

表格

|上下文类型|全局变量|核心作用|生命周期|
|---|---|---|---|
|**应用上下文（App Context）**|`current_app`|指向当前运行的 Flask 应用实例，可访问应用的配置、扩展等|从请求进入时创建，请求处理完成后销毁|
||`g`|全局临时变量，用于在单次请求的多个函数之间传递数据，每次请求都会重置|同应用上下文，单次请求内有效|
|**请求上下文（Request Context）**|`request`|封装了当前 HTTP 请求的所有数据，如请求方法、路径、参数、请求头、请求体、Cookie 等|从请求进入时创建，请求处理完成后销毁|
||`session`|用户会话对象，用于存储用户的会话数据，加密存储在 Cookie 中|同请求上下文|

这四个全局变量可以在视图函数中直接导入使用，无需作为参数传递，示例：

python

运行

```
from flask import Flask, request, session, current_app, g

app = Flask(__name__)
app.secret_key = "your-secret-key"

@app.before_request
def before_request():
    # 请求预处理，将用户信息存入g变量，后续所有函数都可访问
    g.user_id = session.get("user_id")

@app.route("/profile")
def profile():
    # 直接使用request全局变量获取请求参数
    user_id = request.args.get("user_id")
    # 访问g变量中的数据
    current_user_id = g.user_id
    # 访问当前应用实例的配置
    app_name = current_app.name
    return f"用户ID：{user_id}，当前登录用户：{current_user_id}，应用名：{app_name}"
```

#### 2. ThreadLocal 核心原理

在多线程环境下，每个请求对应一个独立的线程，Flask 需要保证每个线程只能访问自己的请求上下文，不能访问其他线程的上下文，这一能力的底层核心是**ThreadLocal（线程本地存储）**。

ThreadLocal 是一种线程安全的存储机制，其核心特性是：**为每个线程维护一个独立的存储副本，每个线程只能读写自己的副本，无法访问其他线程的副本，彻底避免多线程之间的数据竞争**。

Flask 的上下文实现基于 Werkzeug 提供的`Local`对象，是对 ThreadLocal 的增强实现，核心结构包括：

1. **Local 对象**：底层基于 Python 的`threading.local`扩展，为每个线程 / 协程维护独立的存储空间，存储上下文对象；
2. **LocalStack**：基于 Local 实现的栈结构，用于存储上下文对象，支持上下文的入栈、出栈、栈顶获取，Flask 的应用上下文与请求上下文分别存储在两个独立的 LocalStack 中；
3. **LocalProxy**：本地代理对象，`request`、`session`、`current_app`、`g`都是 LocalProxy 代理对象，其核心作用是动态获取 LocalStack 栈顶的上下文对象，对外暴露与原对象完全一致的接口，无需手动处理栈的操作。

#### 3. 上下文的完整生命周期

1. **请求进入**：客户端发起 HTTP 请求，WSGI 服务器调用 Flask 应用的`__call__`方法；
2. **上下文入栈**：创建请求上下文与应用上下文，分别推入对应的 LocalStack 栈中，此时`request`、`current_app`等代理对象可正常访问；
3. **请求预处理**：执行`before_request`钩子函数，完成请求的预处理，如用户认证、权限校验；
4. **视图函数执行**：路由匹配到对应的视图函数，执行业务逻辑，生成响应对象；
5. **响应后处理**：执行`after_request`钩子函数，对响应对象进行后处理，如添加响应头、记录日志；
6. **上下文出栈**：响应返回给客户端后，请求上下文与应用上下文从 LocalStack 栈中弹出，销毁上下文对象，`request`等代理对象无法再访问；
7. **请求结束**：响应传递给 WSGI 服务器，返回给客户端，整个请求流程结束。

#### 4. 离线上下文使用

在非请求场景（如脚本、单元测试）中使用`request`、`current_app`等全局变量时，需要手动创建并推入上下文，示例：

python

运行

```
from flask import current_app
from app import app

# 手动推入应用上下文
with app.app_context():
    # 上下文内可正常访问current_app
    print(current_app.config)
    # 执行数据库操作等依赖应用上下文的逻辑
```

### 9.3.4 Flask 核心扩展生态

Flask 的核心优势在于其丰富的第三方扩展生态，几乎所有 Web 开发所需的能力都有成熟的扩展支持，开发者可按需选择，灵活组装项目架构。

#### 1. SQLAlchemy：Python 最强大的 ORM

SQLAlchemy 是 Python 生态中功能最完善、性能最强大的 ORM 框架，采用 “数据映射器” 模式，相比 Django ORM 更加灵活、可控，支持原生 SQL 的所有能力，同时提供完整的 ORM 映射，是 Flask 生态中数据库操作的事实标准，通过`Flask-SQLAlchemy`扩展与 Flask 无缝集成。

##### （1）核心架构

SQLAlchemy 分为两个核心层级：

- **Core 层**：SQL 表达式语言，底层抽象，提供了与原生 SQL 几乎完全一致的表达能力，可灵活构建复杂 SQL 语句，性能接近原生 SQL；
- **ORM 层**：基于 Core 层的高层对象映射封装，实现 Python 类与数据库表的映射，面向对象的数据库操作，是业务开发的主要使用层级。

##### （2）Flask-SQLAlchemy 基础使用

安装：`pip install flask-sqlalchemy`

基础示例：

python

运行

```
from flask import Flask
from flask_sqlalchemy import SQLAlchemy

# 初始化Flask应用
app = Flask(__name__)
# 数据库配置
app.config["SQLALCHEMY_DATABASE_URI"] = "mysql+pymysql://user:password@localhost:3306/flask_db"
app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False

# 初始化SQLAlchemy实例
db = SQLAlchemy(app)

# 定义模型类，继承db.Model
class User(db.Model):
    __tablename__ = "sys_user"
    # 主键字段
    id = db.Column(db.BigInteger, primary_key=True, comment="主键ID")
    # 字符串字段
    username = db.Column(db.String(32), unique=True, nullable=False, comment="用户名")
    password = db.Column(db.String(128), nullable=False, comment="密码")
    email = db.Column(db.String(64), unique=True, nullable=False, comment="邮箱")
    age = db.Column(db.Integer, comment="年龄")
    is_active = db.Column(db.Boolean, default=True, comment="是否激活")
    created_at = db.Column(db.DateTime, default=db.func.now(), comment="创建时间")

# 一对多关联模型
class Article(db.Model):
    __tablename__ = "article"
    id = db.Column(db.BigInteger, primary_key=True)
    title = db.Column(db.String(128), nullable=False)
    content = db.Column(db.Text, nullable=False)
    # 外键关联用户
    user_id = db.Column(db.BigInteger, db.ForeignKey("sys_user.id"), nullable=False)
    # 反向引用，通过user.articles获取用户的所有文章
    user = db.relationship("User", backref=db.backref("articles", lazy="dynamic"))

# 创建数据库表
with app.app_context():
    db.create_all()
```

##### （3）核心 CRUD 操作

python

运行

```
# 新增
user = User(username="zhangsan", password="123456", email="zhangsan@example.com")
db.session.add(user)
db.session.commit()

# 查询
# 查询所有用户
users = User.query.all()
# 条件查询
user = User.query.filter_by(username="zhangsan").first()
# 复杂条件查询
users = User.query.filter(User.age > 18, User.is_active == True).order_by(User.created_at.desc()).all()
# 分页查询
pagination = User.query.paginate(page=1, per_page=20)

# 更新
user = User.query.get(1)
user.age = 25
db.session.commit()

# 删除
user = User.query.get(1)
db.session.delete(user)
db.session.commit()

# 事务
with db.session.begin():
    db.session.add(User(username="test", password="123456", email="test@example.com"))
    db.session.add(Article(title="测试文章", content="内容", user_id=1))
```

##### （4）核心优势与 Django ORM 对比

表格

|特性|SQLAlchemy|Django ORM|
|---|---|---|
|设计模式|数据映射器，完全解耦模型与数据库表|活动记录，模型与表强绑定|
|灵活性|极高，支持复杂 SQL、自定义查询、联合查询、存储过程|中等，简单场景封装完善，复杂场景需原生 SQL|
|数据库兼容|全数据库兼容，同一套代码适配所有关系型数据库|兼容主流数据库，部分高级特性数据库专属|
|性能|更高，Core 层接近原生 SQL 性能|中等，封装层级更高，有一定性能损耗|
|学习成本|较高，概念多，上手难度大|较低，封装完善，上手简单|
|适用场景|复杂查询、高性能要求、非 Django 项目|Django 项目、中后台管理系统、简单 CRUD 场景|

#### 2. Alembic：数据库迁移工具

Alembic 是 SQLAlchemy 官方开发的数据库迁移工具，对应 Django 的 migrations，用于管理数据库表结构的版本变更，支持自动生成迁移脚本、版本升级、回滚，是 SQLAlchemy 生态的标准迁移方案，通过`Flask-Migrate`扩展与 Flask 无缝集成。

核心常用命令：

bash

运行

```
# 初始化迁移环境
flask db init
# 自动生成迁移脚本
flask db migrate -m "创建用户表"
# 执行迁移脚本，升级数据库
flask db upgrade
# 回滚上一个版本
flask db downgrade
```

#### 3. 其他核心 Flask 扩展

表格

|扩展名称|核心功能|
|---|---|
|Flask-WTF|表单处理、数据验证、CSRF 防护，对应 Django Form|
|Flask-Login|用户会话管理、登录认证、权限控制|
|Flask-JWT-Extended|JWT 认证，前后端分离 API 开发|
|Flask-Caching|缓存支持，兼容 Redis、Memcached、内存缓存|
|Flask-CORS|跨域资源共享支持，解决前后端分离跨域问题|
|Flask-Limiter|接口限流，防止恶意刷取|
|Flask-Session|服务端 Session 支持，兼容 Redis、文件等存储|
|Flask-RESTful / Flask-Smorest|RESTful API 开发支持，对应 Django DRF|

---
