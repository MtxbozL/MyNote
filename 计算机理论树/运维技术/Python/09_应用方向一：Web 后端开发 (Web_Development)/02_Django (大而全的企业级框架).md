
## 9.2 Django 企业级 Web 框架

Django 是 Python 生态中最成熟、应用最广泛的**大而全的企业级 Web 框架**，遵循 “电池内置（Batteries Included）” 的设计哲学，内置了 Web 开发所需的几乎所有组件，无需开发者重复造轮子，同时遵循 DRY（Don't Repeat Yourself）设计原则，通过 ORM、Admin 后台、表单系统、权限体系等核心能力，大幅提升中大型 Web 项目的开发效率，是政务、金融、电商、内容管理等企业级场景的首选框架。

### 9.2.1 MVT 架构模型与请求生命周期

#### 1. MVT 架构核心定义

Django 采用 MVT 分层架构，是传统 MVC 架构的 Python 化变体，实现了数据、业务逻辑、视图的解耦，各层职责清晰明确：

表格

|架构层|全称|核心职责|对应 MVC 架构层级|
|---|---|---|---|
|Model|模型层|负责数据结构定义、数据库映射、数据 CRUD 操作、业务规则封装，是项目的核心数据层|Model（模型层）|
|View|视图层|负责核心业务逻辑处理，接收请求，调用 Model 层获取 / 处理数据，将数据传递给 Template 层，返回响应给客户端|Controller（控制层）|
|Template|模板层|负责页面渲染、前端展示，接收 View 层传递的数据，生成最终的 HTML 页面返回给客户端|View（视图层）|

除核心三层外，Django 还包含两个核心调度层：

- **URL 路由层（URLconf）**：负责 URL 路径与视图函数的映射，将客户端的请求分发到对应的视图处理；
- **中间件（Middleware）**：请求 / 响应的全局钩子，可在请求到达视图前、响应返回客户端前执行统一的预处理 / 后处理逻辑，如认证、日志、跨域处理、缓存等。

#### 2. Django 请求生命周期【面试超高频考点】

Django 处理一个 HTTP 请求的完整流程，是理解框架运行原理的核心，完整执行步骤如下：

1. **Web 服务器接收请求**：客户端发起的 HTTP 请求，由 WSGI/ASGI 服务器（Gunicorn/Uvicorn）接收，解析为符合协议规范的请求数据，传递给 Django 核心；
2. **中间件预处理请求**：请求依次经过所有中间件的`process_request`方法，执行全局预处理，如 Session 加载、用户认证、CSRF 校验、请求日志记录等，任何一个中间件返回响应，都会直接终止流程，进入响应处理阶段；
3. **URL 路由匹配**：Django 加载项目的 URLconf 配置，将请求的路径与路由规则逐一匹配，找到对应的视图函数 / 类视图，匹配失败则返回 404 响应；
4. **视图业务逻辑处理**：执行匹配到的视图函数 / 类视图，核心流程包括：
    
    - 解析请求参数、请求体，进行数据验证；
    - 调用 Model 层的 ORM 接口，完成数据库的增删改查操作；
    - 渲染 Template 模板，或构建 JSON 响应数据；
    - 生成 HttpResponse 响应对象；
    
5. **中间件后处理响应**：生成的响应对象，逆序经过所有中间件的`process_response`方法，执行全局后处理，如响应头修改、缓存设置、跨域头添加、响应日志记录等；
6. **服务器返回响应**：处理完成的响应对象，传递给 WSGI/ASGI 服务器，封装为标准 HTTP 响应，返回给客户端，整个请求流程结束。

### 9.2.2 Django ORM 数据库映射

ORM（Object-Relational Mapping，对象关系映射）是 Django 的核心灵魂，将关系型数据库的表结构映射为 Python 的类（模型类），表的行映射为类的实例，开发者无需编写原生 SQL，通过面向对象的 Python 语法即可完成数据库操作，彻底屏蔽了不同数据库的 SQL 语法差异，同时内置了防 SQL 注入、事务管理、关联查询等核心能力。

#### 1. 模型类定义与字段类型

Django 的模型类必须继承`django.db.models.Model`，类的属性对应数据库表的字段，类的 Meta 内部类定义表的元信息（表名、索引、排序等）。

示例：标准模型类定义

python

运行

```
from django.db import models
from django.utils import timezone

class User(models.Model):
    """用户模型类，映射数据库中的user表"""
    # 主键字段：不指定主键时，Django自动生成id自增主键
    id = models.BigAutoField(primary_key=True, verbose_name="主键ID")
    # 字符串字段，对应数据库varchar类型
    username = models.CharField(max_length=32, unique=True, verbose_name="用户名")
    password = models.CharField(max_length=128, verbose_name="密码")
    email = models.EmailField(max_length=64, unique=True, verbose_name="邮箱")
    # 整数字段
    age = models.IntegerField(null=True, blank=True, verbose_name="年龄")
    # 布尔字段
    is_active = models.BooleanField(default=True, verbose_name="是否激活")
    # 日期时间字段
    created_at = models.DateTimeField(default=timezone.now, verbose_name="创建时间")
    updated_at = models.DateTimeField(auto_now=True, verbose_name="更新时间")

    class Meta:
        # 数据库表名，默认格式：app名_模型类名小写
        db_table = "sys_user"
        # 表注释
        verbose_name = "用户表"
        verbose_name_plural = verbose_name
        # 排序规则
        ordering = ["-created_at"]
        # 联合唯一索引
        unique_together = [("username", "email")]
        # 普通索引
        indexes = [
            models.Index(fields=["created_at"]),
        ]

    def __str__(self):
        return self.username
```

Django ORM 提供了全场景的字段类型，覆盖所有数据库字段，核心高频字段包括：

- 字符串类：`CharField`、`TextField`、`EmailField`、`URLField`、`SlugField`；
- 数值类：`IntegerField`、`BigIntegerField`、`FloatField`、`DecimalField`（高精度十进制，适用于金额）；
- 日期时间类：`DateField`、`DateTimeField`、`TimeField`；
- 布尔类：`BooleanField`、`NullBooleanField`；
- 文件类：`FileField`、`ImageField`；
- 关系类：`ForeignKey`（一对多）、`OneToOneField`（一对一）、`ManyToManyField`（多对多）。

#### 2. 关联关系映射

Django ORM 原生支持关系型数据库的三种核心关联关系，自动处理外键约束、关联查询：

1. **一对多关系**：通过`ForeignKey`字段实现，外键字段定义在 “多” 的一方，示例：
    
    python
    
    运行
    
    ```
    # 一的一方：文章分类
    class Category(models.Model):
        name = models.CharField(max_length=32, verbose_name="分类名称")
    
    # 多的一方：文章，一个分类下有多篇文章
    class Article(models.Model):
        title = models.CharField(max_length=128, verbose_name="标题")
        content = models.TextField(verbose_name="内容")
        # 外键关联分类，on_delete定义级联删除规则
        category = models.ForeignKey(Category, on_delete=models.CASCADE, verbose_name="所属分类")
    ```
    
    核心级联删除规则：
    
    - `CASCADE`：级联删除，主表数据删除时，关联的从表数据同步删除；
    - `PROTECT`：保护模式，存在关联数据时禁止删除主表数据；
    - `SET_NULL`：主表数据删除时，从表外键字段设为 NULL（需配合`null=True`）；
    - `SET_DEFAULT`：主表数据删除时，从表外键字段设为默认值（需配合`default`）。
    
2. **一对一关系**：通过`OneToOneField`字段实现，本质是带唯一约束的外键，适用于表的垂直拆分，示例：
    
    python
    
    运行
    
    ```
    class User(models.Model):
        username = models.CharField(max_length=32, verbose_name="用户名")
    
    class UserProfile(models.Model):
        # 一对一关联用户，用户删除时，资料同步删除
        user = models.OneToOneField(User, on_delete=models.CASCADE, verbose_name="关联用户")
        avatar = models.URLField(verbose_name="头像")
        phone = models.CharField(max_length=11, verbose_name="手机号")
    ```
    
3. **多对多关系**：通过`ManyToManyField`字段实现，Django 自动生成中间关联表，无需手动定义，示例：
    
    python
    
    运行
    
    ```
    class Article(models.Model):
        title = models.CharField(max_length=128, verbose_name="标题")
        content = models.TextField(verbose_name="内容")
        # 多对多关联标签，一篇文章有多个标签，一个标签对应多篇文章
        tags = models.ManyToManyField("Tag", verbose_name="标签")
    
    class Tag(models.Model):
        name = models.CharField(max_length=16, verbose_name="标签名称")
    ```
    

#### 3. ORM 核心 CRUD 操作

Django ORM 的所有数据库操作均通过模型类的`objects`管理器（Manager）实现，核心操作如下：

##### （1）新增数据（Create）

python

运行

```
# 方式1：create()方法，一步创建并保存到数据库
user = User.objects.create(username="zhangsan", password="123456", email="zhangsan@example.com")

# 方式2：实例化后save()方法，先创建实例，再保存到数据库
user = User(username="lisi", password="123456", email="lisi@example.com")
user.age = 20
user.save()

# 批量创建，大幅提升大量数据插入性能，仅触发一次数据库交互
User.objects.bulk_create([
    User(username="user1", password="123456", email="user1@example.com"),
    User(username="user2", password="123456", email="user2@example.com"),
])
```

##### （2）查询数据（Retrieve）

Django ORM 的查询核心是**QuerySet**，是惰性求值的查询集，仅当真正使用数据时才会执行数据库查询，支持链式调用，核心查询 API 如下：

表格

|API|功能描述|返回值类型|
|---|---|---|
|`all()`|查询表中所有数据|QuerySet|
|`get(**kwargs)`|查询单条数据，匹配多条 / 无数据时抛出异常|模型实例|
|`filter(**kwargs)`|过滤符合条件的数据，返回多条|QuerySet|
|`exclude(**kwargs)`|排除符合条件的数据，返回剩余数据|QuerySet|
|`order_by(*fields)`|对查询结果排序，`-字段名`表示降序|QuerySet|
|`values(*fields)`|返回字典格式的查询结果，仅包含指定字段|QuerySet[dict]|
|`values_list(*fields, flat=False)`|返回元组格式的查询结果，flat=True 时返回单字段的列表|QuerySet[tuple]|
|`count()`|统计查询结果的数量，返回整数|int|
|`exists()`|判断查询结果是否存在，返回布尔值|bool|
|`first()` / `last()`|返回查询结果的第一条 / 最后一条数据|模型实例 / None|
|`distinct()`|对查询结果去重|QuerySet|

**查询条件表达式**：通过`字段名__条件名`实现复杂查询，核心高频条件：

python

运行

```
# 精确匹配，等价于=
User.objects.filter(username="zhangsan")
# 模糊查询，包含指定字符串，等价于LIKE '%xxx%'
User.objects.filter(username__contains="zhang")
# 大于/大于等于/小于/小于等于，等价于>/>=/</<=
User.objects.filter(age__gt=18)
User.objects.filter(age__gte=18)
User.objects.filter(age__lt=30)
User.objects.filter(age__lte=30)
# 范围查询，等价于IN
User.objects.filter(id__in=[1,2,3])
# 区间查询，等价于BETWEEN
User.objects.filter(created_at__range=["2024-01-01", "2024-12-31"])
# 为空判断，等价于IS NULL
User.objects.filter(age__isnull=True)
```

**关联查询**：通过`外键字段名__关联表字段名`实现跨表关联查询，示例：

python

运行

```
# 查询分类名称为"技术"的所有文章
Article.objects.filter(category__name="技术")
# 查询用户名为zhangsan的用户资料
UserProfile.objects.filter(user__username="zhangsan")
```

##### （3）更新数据（Update）

python

运行

```
# 方式1：单条数据更新，修改实例属性后save()
user = User.objects.get(id=1)
user.age = 25
user.save()

# 方式2：批量更新，直接过滤后update()，仅触发一次数据库交互，性能更高
User.objects.filter(age__isnull=True).update(age=18)

# F()表达式：基于原有字段值更新，避免竞态条件，无需先查询再更新
from django.db.models import F
# 所有用户的年龄+1
User.objects.all().update(age=F("age") + 1)
```

##### （4）删除数据（Delete）

python

运行

```
# 方式1：单条数据删除
user = User.objects.get(id=1)
user.delete()

# 方式2：批量删除，过滤后直接delete()
User.objects.filter(is_active=False).delete()
```

#### 4. QuerySet 核心特性【面试高频考点】

1. **惰性求值**：QuerySet 在创建、链式调用时，不会执行任何数据库查询，仅当对 QuerySet 进行迭代、切片、取值、len ()、list ()、bool () 等操作时，才会真正执行 SQL 查询，大幅减少不必要的数据库交互；
2. **缓存机制**：QuerySet 执行一次查询后，会将结果缓存到内存中，后续对该 QuerySet 的再次访问，不会重复查询数据库，直接使用缓存数据，提升性能；
3. **链式调用**：QuerySet 的所有过滤、排序、去重方法都会返回新的 QuerySet，可无限链式调用，代码简洁易读；
4. **可迭代性**：QuerySet 实现了迭代器协议，可直接用于 for 循环遍历。

#### 5. 事务管理

Django ORM 原生支持数据库事务，通过`django.db.transaction`模块实现，核心用法：

1. **装饰器用法**：整个函数内的所有数据库操作在同一个事务中执行
    
    python
    
    运行
    
    ```
    from django.db import transaction
    
    @transaction.atomic
    def create_user_and_profile():
        user = User.objects.create(username="test", password="123456", email="test@example.com")
        UserProfile.objects.create(user=user, phone="13800138000")
    ```
    
2. **上下文管理器用法**：指定代码块在事务中执行
    
    python
    
    运行
    
    ```
    with transaction.atomic():
        user = User.objects.create(username="test2", password="123456", email="test2@example.com")
        UserProfile.objects.create(user=user, phone="13800138001")
    ```
    
3. **事务回滚点**：支持设置保存点，实现部分回滚，不影响整个事务。

### 9.2.3 Admin 后台管理系统

Admin 后台是 Django 最具特色的核心能力，是基于 Django ORM 自动生成的可视化后台管理系统，无需编写前端代码，仅需简单配置，即可实现模型的增删改查、搜索、筛选、权限控制等管理功能，大幅降低企业级后台的开发成本。

#### 1. Admin 基础使用

1. **模型注册**：在应用的`admin.py`文件中注册模型，即可在 Admin 后台展示
    
    python
    
    运行
    
    ```
    from django.contrib import admin
    from .models import User, Article, Category
    
    # 最简注册方式
    admin.site.register(User)
    admin.site.register(Category)
    admin.site.register(Article)
    ```
    
2. **Admin 后台访问**：创建超级管理员用户`python manage.py createsuperuser`，启动项目后访问`/admin`路径，登录后即可进入管理后台。

#### 2. ModelAdmin 高级配置

通过继承`admin.ModelAdmin`类，可对 Admin 后台的展示、交互、功能进行全量定制，示例：

python

运行

```
from django.contrib import admin
from .models import Article

@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):
    # 列表页展示的字段
    list_display = ("id", "title", "category", "is_published", "created_at")
    # 列表页可点击进入详情页的字段
    list_display_links = ("id", "title")
    # 列表页右侧的筛选器
    list_filter = ("category", "is_published", "created_at")
    # 列表页顶部的搜索框，支持模糊搜索的字段
    search_fields = ("title", "content")
    # 列表页可直接编辑的字段，无需进入详情页
    list_editable = ("is_published",)
    # 排序规则
    ordering = ("-created_at",)
    # 详情页分组展示的字段
    fieldsets = (
        ("基础信息", {"fields": ("title", "category", "tags")}),
        ("内容", {"fields": ("content",)}),
        ("发布设置", {"fields": ("is_published", "publish_time")}),
    )
    # 多对多字段的选择控件
    filter_horizontal = ("tags",)
    # 每页展示的数据条数
    list_per_page = 20
    # 日期分层导航
    date_hierarchy = "created_at"
```

#### 3. Admin 权限体系

Django Admin 内置了完善的基于 RBAC 的权限控制体系，核心包括：

- **用户（User）**：后台登录的用户，分为超级管理员与普通用户；
- **组（Group）**：用户的分组，可批量为组分配权限，组内的用户自动继承组的所有权限；
- **权限（Permission）**：Django 为每个模型自动生成 4 个核心权限：新增、修改、删除、查看，可自定义权限，精确控制用户对模型的操作权限。

超级管理员拥有所有权限，普通用户仅能访问被分配的模型与操作，可实现精细化的后台权限管控，满足企业级多角色管理需求。

### 9.2.4 Form 表单系统

Django Form 表单系统是服务端渲染场景下的核心组件，负责表单的渲染、数据验证、错误处理、数据保存，彻底解决了表单开发中的重复代码与安全问题。

Django 提供两种核心表单类型：

1. **Form**：基础表单，用于自定义字段与验证规则，与模型解耦；
2. **ModelForm**：与模型绑定的表单，自动从模型生成表单字段、验证规则，支持数据的自动保存，是最常用的表单类型。

#### 1. ModelForm 示例

python

运行

```
from django import forms
from .models import User

class UserRegisterForm(forms.ModelForm):
    # 扩展表单字段，模型中不存在的字段
    password_confirm = forms.CharField(max_length=128, widget=forms.PasswordInput, label="确认密码")

    class Meta:
        # 绑定的模型类
        model = User
        # 表单包含的字段，__all__表示所有字段
        fields = ("username", "email", "password", "password_confirm", "age")
        # 字段的渲染控件
        widgets = {
            "password": forms.PasswordInput,
            "age": forms.NumberInput,
        }
        # 字段的标签文本
        labels = {
            "username": "用户名",
            "email": "邮箱",
            "password": "密码",
            "age": "年龄",
        }
        # 字段的错误提示信息
        error_messages = {
            "username": {
                "required": "用户名不能为空",
                "unique": "用户名已存在",
            },
            "email": {
                "required": "邮箱不能为空",
                "invalid": "邮箱格式不正确",
            },
        }

    # 自定义字段验证方法：clean_字段名
    def clean_username(self):
        username = self.cleaned_data.get("username")
        if len(username) < 3:
            raise forms.ValidationError("用户名长度不能少于3位")
        return username

    # 全局表单验证方法，校验多字段关联
    def clean(self):
        cleaned_data = super().clean()
        password = cleaned_data.get("password")
        password_confirm = cleaned_data.get("password_confirm")
        if password and password_confirm and password != password_confirm:
            raise forms.ValidationError("两次输入的密码不一致")
        return cleaned_data
```

#### 2. 表单核心能力

- **自动渲染**：表单实例可直接在模板中渲染，支持完整表单、单个字段渲染，自动生成 HTML 表单控件；
- **数据验证**：自动执行字段类型验证、长度验证、唯一性验证、自定义规则验证，验证失败自动生成错误信息；
- **错误处理**：自动携带验证失败的错误信息，在模板中展示给用户；
- **数据保存**：ModelForm 的`save()`方法可直接将验证通过的数据保存到数据库，支持新增与更新。

### 9.2.5 Django REST Framework（DRF）

Django REST Framework（DRF）是基于 Django 的 RESTful API 开发框架，是 Django 生态中前后端分离开发的事实标准，在 Django ORM 的基础上，提供了序列化器、视图集、认证权限、分页限流等全链路 API 开发能力，大幅简化 RESTful API 的开发流程。

#### 1. 核心组件：序列化器（Serializer）

序列化器是 DRF 的核心灵魂，承担三大核心职责：

1. **序列化**：将 Django 模型实例、QuerySet 转换为 JSON 格式的数据，返回给前端；
2. **反序列化**：将前端传入的 JSON 数据进行解析、验证，转换为 Python 对象；
3. **数据验证**：提供完整的字段验证、自定义验证规则，验证失败返回标准化的错误信息。

DRF 提供两种核心序列化器：

- **Serializer**：基础序列化器，需手动定义所有字段与验证规则；
- **ModelSerializer**：与模型绑定的序列化器，自动从模型生成字段、验证规则，简化开发。

示例：ModelSerializer 定义

python

运行

```
from rest_framework import serializers
from .models import Article, Category

class CategorySerializer(serializers.ModelSerializer):
    class Meta:
        model = Category
        fields = ("id", "name")

class ArticleSerializer(serializers.ModelSerializer):
    # 嵌套序列化关联分类数据
    category = CategorySerializer(read_only=True)
    # 分类ID，用于创建/更新时传入
    category_id = serializers.IntegerField(write_only=True)

    class Meta:
        model = Article
        # 序列化包含的字段
        fields = ("id", "title", "content", "category", "category_id", "is_published", "created_at")
        # 只读字段，前端不可修改
        read_only_fields = ("id", "created_at")
        # 必填字段
        required_fields = ("title", "content", "category_id")

    # 自定义字段验证
    def validate_title(self, value):
        if len(value) < 5:
            raise serializers.ValidationError("文章标题长度不能少于5位")
        return value

    # 全局验证
    def validate(self, attrs):
        if attrs.get("is_published") and not attrs.get("content"):
            raise serializers.ValidationError("发布的文章必须填写内容")
        return attrs
```

#### 2. 视图与视图集

DRF 提供了多层级的视图封装，从底层到高层逐步简化 API 开发，核心分为三类：

1. **APIView**：DRF 的基础视图类，继承自 Django 的 View，封装了请求解析、响应渲染、认证权限、异常处理等核心能力，支持按 HTTP 请求方法定义对应的处理函数（get/post/put/delete 等），灵活性最高。
    
2. **GenericAPIView + Mixin 扩展类**：GenericAPIView 是通用视图基类，封装了 ORM 查询、序列化器管理等通用逻辑，配合 5 个 Mixin 扩展类，实现 CRUD 操作的复用：
    
    - `ListModelMixin`：实现列表查询`list()`方法；
    - `RetrieveModelMixin`：实现单条详情查询`retrieve()`方法；
    - `CreateModelMixin`：实现新增`create()`方法；
    - `UpdateModelMixin`：实现更新`update()`方法；
    - `DestroyModelMixin`：实现删除`destroy()`方法。
    
3. **视图集（ViewSet）**：DRF 的最高层封装，将同一资源的所有 CRUD 操作封装在一个类中，无需手动定义多个视图，配合 Router 自动生成 URL 路由，是 DRF 开发的首选方案。核心视图集包括：
    
    - `ModelViewSet`：默认集成了所有 5 个 Mixin 扩展类，提供完整的增删改查能力；
    - `ReadOnlyModelViewSet`：仅集成列表与详情查询 Mixin，提供只读接口。
    

示例：ModelViewSet 定义

python

运行

```
from rest_framework.viewsets import ModelViewSet
from rest_framework.permissions import IsAuthenticated
from .models import Article
from .serializers import ArticleSerializer

class ArticleViewSet(ModelViewSet):
    # 查询集
    queryset = Article.objects.all().order_by("-created_at")
    # 序列化器类
    serializer_class = ArticleSerializer
    # 权限控制：仅登录用户可访问
    permission_classes = [IsAuthenticated]

    # 重写查询集，实现自定义过滤
    def get_queryset(self):
        queryset = super().get_queryset()
        # 仅返回已发布的文章
        return queryset.filter(is_published=True)
```

#### 3. 路由自动注册（Router）

DRF 的 Router 可自动为视图集生成 RESTful 风格的 URL 路由，无需手动编写 URLconf，示例：

python

运行

```
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import ArticleViewSet, CategoryViewSet

# 创建路由器
router = DefaultRouter()
# 注册视图集
router.register(r"articles", ArticleViewSet)
router.register(r"categories", CategoryViewSet)

# 生成的路由：
# /articles/ → 列表查询/新增
# /articles/{id}/ → 详情查询/更新/删除

urlpatterns = [
    path("api/", include(router.urls)),
]
```

#### 4. 认证与权限体系

DRF 提供了完整的认证与权限框架，实现接口的访问控制，核心包括：

1. **认证类（Authentication）**：验证用户身份，确认请求的发起者是谁，内置认证类包括：
    
    - `SessionAuthentication`：基于 Django Session 的认证，适用于前后端同域场景；
    - `TokenAuthentication`：基于 Token 的简单认证；
    - `JWT认证`：通过`djangorestframework-simplejwt`扩展实现，是前后端分离场景的主流方案，基于 JSON Web Token 实现无状态认证，支持过期时间、刷新 Token、自定义载荷等能力。
    
2. **权限类（Permission）**：在身份认证通过后，判断用户是否有权限访问该接口、执行对应操作，内置权限类包括：
    
    - `AllowAny`：允许所有用户访问，无权限限制；
    - `IsAuthenticated`：仅允许登录用户访问；
    - `IsAdminUser`：仅允许管理员用户访问；
    - `IsAuthenticatedOrReadOnly`：登录用户可执行所有操作，匿名用户仅可执行 GET/HEAD/OPTIONS 等只读操作；
    - 支持自定义权限类，实现精细化的接口权限控制。
    

#### 5. 其他核心组件

- **分页**：内置多种分页器，如`PageNumberPagination`、`LimitOffsetPagination`，实现列表数据的分页返回；
- **过滤与排序**：通过`django-filter`扩展实现复杂的条件过滤，支持字段排序、搜索过滤；
- **限流**：内置限流类，限制接口的访问频率，防止接口被恶意刷取，保护服务稳定性；
- **版本控制**：支持多种 API 版本控制方案，实现接口的多版本兼容；
- **异常处理**：统一的异常处理机制，自动将异常转换为标准化的 JSON 响应；
- **自动文档**：内置`coreapi`与`drf-yasg`扩展，自动生成 Swagger API 文档。

---