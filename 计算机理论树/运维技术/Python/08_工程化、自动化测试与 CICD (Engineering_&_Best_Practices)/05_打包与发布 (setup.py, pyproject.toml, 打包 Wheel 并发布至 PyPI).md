
Python 项目的打包与发布，是将开发完成的代码打包为可分发、可安装的标准格式，发布到 PyPI（Python Package Index）公共仓库或私有仓库，供其他开发者安装使用。Python 现代打包标准遵循 PEP 621 规范，基于`pyproject.toml`配置文件，使用`build`模块构建，`twine`工具发布，替代传统的`setup.py`方案。

### 8.5.1 核心概念与打包格式

Python 支持两种核心打包格式，分别为源码包与 Wheel 包，其中 Wheel 包是现代推荐的分发格式。

|格式|后缀名|核心描述|优势|劣势|
|---|---|---|---|---|
|源码包（sdist）|`.tar.gz`|包含项目的完整源代码、构建脚本、配置文件，安装时需要在本地执行构建流程|兼容性强，适配所有平台，可查看完整源码|安装速度慢，需要本地构建环境，可能需要编译 C 扩展|
|Wheel 包|`.whl`|预构建的二进制包，是 Python 的官方标准二进制分发格式，安装时无需构建，直接解压即可|安装速度极快，无需本地构建环境，支持平台特定的预编译扩展，可区分纯 Python 包与平台相关包|不同平台、不同 Python 版本需要构建对应的 Wheel 包|

**核心规范**：纯 Python 项目（无 C 扩展）可构建通用的`any`平台 Wheel 包，跨平台兼容；包含 C/C++ 扩展的项目，需要针对不同平台、不同 Python 版本构建对应的平台特定 Wheel 包。

### 8.5.2 项目结构规范

可发布的 Python 项目必须遵循标准的目录结构，保证打包工具能正确识别项目结构，示例如下：

```bash
my_python_package/
├── my_project/                    # 项目核心包目录，包名与项目名一致
│   ├── __init__.py                # 包初始化文件，定义对外暴露的接口
│   ├── core.py                    # 核心模块
│   ├── utils.py                   # 工具模块
│   └── __main__.py                # 命令行入口（可选）
├── tests/                         # 测试用例目录
│   ├── __init__.py
│   ├── test_core.py
│   └── test_utils.py
├── docs/                          # 项目文档
├── examples/                      # 使用示例
├── pyproject.toml                 # 项目核心配置文件，打包、依赖、元信息
├── README.md                      # 项目说明文档，必须包含
├── LICENSE                        # 开源协议文件，必须包含
├── .gitignore                     # Git忽略文件
└── CHANGELOG.md                   # 版本更新日志
```

### 8.5.3 pyproject.toml 打包配置

`pyproject.toml`是 PEP 621 规定的 Python 项目打包标准配置文件，替代了传统的`setup.py`、`setup.cfg`、`MANIFEST.in`等文件，Poetry、Flit、Setuptools 等所有现代打包工具均兼容该规范。

#### 1. 基于 Setuptools 的标准打包配置

适用于不使用 Poetry 的原生打包场景，完整配置如下：

```toml
# 构建系统配置，指定构建后端
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

# 项目元信息，遵循PEP 621规范
[project]
name = "my-python-package"  # 包名，PyPI上唯一，小写字母+连字符
version = "0.1.0"           # 版本号，遵循语义化版本规范
authors = [
  { name="Your Name", email="your@email.com" }
]
maintainers = [
  { name="Your Name", email="your@email.com" }
]
description = "一个Python示例包"  # 包的简短描述
readme = "README.md"              # 详细说明文档，支持Markdown
license = { file="LICENSE" }      # 开源协议
requires-python = ">=3.8"          # 支持的Python版本
keywords = ["python", "demo", "example"]  # 关键词，用于PyPI搜索
classifiers = [                     # 分类器，用于PyPI分类筛选
  "Development Status :: 3 - Alpha",
  "Intended Audience :: Developers",
  "License :: OSI Approved :: MIT License",
  "Operating System :: OS Independent",
  "Programming Language :: Python :: 3",
  "Programming Language :: Python :: 3.8",
  "Programming Language :: Python :: 3.9",
  "Programming Language :: Python :: 3.10",
  "Programming Language :: Python :: 3.11",
  "Programming Language :: Python :: 3.12",
  "Programming Language :: Python :: 3.13",
  "Topic :: Software Development :: Libraries :: Python Modules"
]
# 项目依赖，生产环境
dependencies = [
  "requests>=2.31.0",
  "pydantic>=2.0.0"
]
# 可选依赖，分组安装
[project.optional-dependencies]
dev = [
  "pytest>=8.0.0",
  "black>=24.0.0",
  "flake8>=6.0.0",
  "build>=1.0.0",
  "twine>=4.0.0"
]
# 命令行入口点，安装后可直接在终端执行命令
[project.scripts]
my-project-cli = "my_project.__main__:main"

# 项目URL，用于PyPI页面展示
[project.urls]
Homepage = "https://github.com/yourname/my-python-package"
Documentation = "https://my-python-package.readthedocs.io/"
Repository = "https://github.com/yourname/my-python-package.git"
Bug Tracker = "https://github.com/yourname/my-python-package/issues"
```

#### 2. 基于 Poetry 的打包配置

使用 Poetry 管理的项目，无需额外配置，`pyproject.toml`已包含所有打包所需的元信息，Poetry 原生支持打包命令，无需额外安装构建工具。

### 8.5.4 打包构建流程

#### 1. 环境准备

安装打包所需工具：

```bash
# 原生打包工具
pip install build twine
# 若使用Poetry，无需安装上述工具
```

#### 2. 执行打包

- **原生 Setuptools 方案**：在项目根目录执行构建命令，自动生成源码包与 Wheel 包，输出到`dist/`目录：

    ```bash
    python -m build
    ```
    
- **Poetry 方案**：执行 Poetry 打包命令，自动构建源码包与 Wheel 包到`dist/`目录：

    ```bash
    poetry build
    ```

构建完成后，`dist/`目录会生成两个文件：

- 源码包：`my-python-package-0.1.0.tar.gz`
- 纯 Python Wheel 包：`my_python_package-0.1.0-py3-none-any.whl`

#### 3. 本地验证

打包完成后，可在本地安装验证包是否正常：

```bash
# 安装本地构建的Wheel包
pip install dist/my_python_package-0.1.0-py3-none-any.whl

# 验证导入与功能
python -c "import my_project; print(my_project.__version__)"

# 验证命令行工具
my-project-cli --help
```

### 8.5.5 发布到 PyPI

PyPI（Python Package Index）是 Python 官方的公共包仓库，`pip install`默认从该仓库拉取包，发布流程如下：

#### 1. 注册账号

- 访问
    
    PyPI 官网，注册账号并完成邮箱验证；
- 生产环境发布前，推荐先发布到
    
    TestPyPI
    测试，TestPyPI 是 PyPI 的测试环境，与正式环境隔离，需单独注册账号。

#### 2. 配置 PyPI 认证

推荐使用 API Token 进行认证，避免使用账号密码，提升安全性：

- 登录 PyPI，进入账户设置，创建 API Token，权限选择整个账户或指定项目；
- 配置认证信息：在用户目录创建`~/.pypirc`文件，写入配置：

    ```bash
    [pypi]
      username = __token__
      password = pypi-你的API Token
    [testpypi]
      repository = https://test.pypi.org/legacy/
      username = __token__
      password = pypi-你的TestPyPI API Token
    ```
    

#### 3. 上传发布

- **先发布到 TestPyPI 测试**：

    ```bash
    # 原生twine上传
    twine upload --repository testpypi dist/*
    # Poetry上传
    poetry publish -r testpypi
    ```
    
    测试安装：`pip install --index-url https://test.pypi.org/simple/ my-python-package`
    
- **正式发布到 PyPI**：

    ```bash
    # 原生twine上传，自动使用.pypirc中的配置
    twine upload dist/*
    # Poetry上传
    poetry publish
    ```
    

发布完成后，即可在 PyPI 官网搜索到你的包，其他开发者可通过`pip install my-python-package`安装使用。

### 8.5.6 打包发布最佳实践

1. **版本号遵循语义化版本规范**：版本号格式为`主版本号.次版本号.补丁版本号`，主版本号：不兼容的 API 变更；次版本号：向后兼容的功能新增；补丁版本号：向后兼容的问题修复；
2. **必填文件必须完整**：`README.md`、`LICENSE`、`CHANGELOG.md`必须包含，`README.md`需清晰说明包的功能、安装方式、使用示例，是用户了解包的核心文档；
3. **开源协议明确**：必须选择合适的开源协议（MIT、Apache 2.0、GPL 等），并包含完整的 LICENSE 文件，明确版权与使用权限；
4. **分类器完整准确**：`classifiers`必须准确填写，包括支持的 Python 版本、操作系统、开发状态、适用场景，便于用户在 PyPI 上搜索筛选；
5. **发布前必须测试**：发布前必须完成本地安装测试、功能测试，先发布到 TestPyPI 验证，再发布到正式 PyPI，避免发布有问题的版本；
6. **禁止重复发布相同版本**：PyPI 不允许覆盖已发布的版本，即使删除了旧版本，也无法重新发布相同版本号的包，版本号必须严格递增；
7. **命名规范**：包名必须小写，使用连字符`-`分隔单词，PyPI 上全局唯一，避免与已有包重名；
8. **最小化发布内容**：通过`.gitignore`、`MANIFEST.in`排除测试、文档、构建缓存等无关文件，减小包的体积。

---

## 本章核心考点与学习要求

1. 熟练掌握 PEP 8 规范的核心强制要求，包括缩进、命名、导入、表达式、注释规范，能写出符合官方标准的 Python 代码；
2. 熟练使用 black、isort、flake8 等代码规范工具，能通过 pre-commit 实现代码提交前的自动校验，保证团队代码规范的落地；
3. 深刻理解传统 requirements.txt 的核心缺陷，熟练使用 Poetry 实现项目的依赖管理、虚拟环境管理，掌握 pyproject.toml 的核心配置、版本约束、依赖分组，能解决依赖冲突问题；
4. 熟练使用 unittest 框架编写单元测试用例，掌握 setUp/tearDown、断言方法、异常断言的用法；
5. 熟练掌握 pytest 测试框架的核心用法，包括测试用例编写、执行参数、fixture 机制（作用域、依赖注入、conftest.py 全局共享）、参数化测试，能基于 pytest 编写高效、可复用的测试用例；
6. 熟练使用 unittest.mock 实现 Mock 测试，掌握 patch 装饰器 / 上下文管理器的用法，能模拟外部依赖、验证调用情况、模拟异常，实现测试隔离；
7. 深刻理解 logging 模块的四大核心组件、日志级别，熟练使用 dictConfig 实现工程化的日志配置，掌握日志轮转、多 Handler 分离、模块化日志器的最佳实践，能规范记录异常日志与业务上下文；
8. 掌握 Python 打包的核心格式、标准项目结构，熟练基于 pyproject.toml 配置打包信息，能完成源码包与 Wheel 包的构建、本地验证、发布到 PyPI 的全流程；
9. 能将本章的工程化规范应用到 Python 项目开发中，搭建企业级的 Python 项目工程化体系，解决团队协作、质量保障、依赖管理、可观测性、分发发布的核心工程问题。