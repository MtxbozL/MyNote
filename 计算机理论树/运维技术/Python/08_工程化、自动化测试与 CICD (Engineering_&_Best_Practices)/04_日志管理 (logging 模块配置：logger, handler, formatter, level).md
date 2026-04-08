
日志是 Python 项目可观测性的核心基础，是线上问题排查、业务监控、审计回溯的核心依据。Python 内置的`logging`模块提供了完整、灵活的日志管理体系，替代了简单的`print()`打印，支持分级日志、多输出渠道、格式化、日志轮转、结构化日志等企业级能力。

### 8.4.1 print () 与 logging 的核心区别

|特性|print()|logging 模块|
|---|---|---|
|日志分级|无分级，所有输出无区别|支持 5 个日志级别，可灵活控制输出粒度|
|输出渠道|仅能输出到控制台|支持同时输出到控制台、文件、网络、日志服务等多渠道|
|格式化|手动拼接格式，无统一规范|支持自定义格式化，自动包含时间、文件名、行号、级别等上下文信息|
|生产环境适配|生产环境无法关闭，大量打印导致性能损耗|生产环境可调整日志级别，关闭调试日志，性能开销极低|
|异常追踪|无法自动记录异常栈信息|支持自动记录完整的异常栈，便于问题排查|
|线程安全|非线程安全，多线程场景下打印内容错乱|原生线程安全，多线程 / 多进程场景下无错乱问题|

### 8.4.2 logging 核心架构与日志级别

#### 1. 日志级别

logging 模块定义了 5 个标准日志级别，优先级从低到高排序，设置日志级别后，仅会输出优先级大于等于设置级别的日志。

|级别|数值|适用场景|
|---|---|---|
|`DEBUG`|10|调试信息，开发环境使用，记录程序执行的详细流程、变量值，用于问题定位|
|`INFO`|20|常规信息，记录程序的正常执行流程，如服务启动、任务完成、业务状态变更|
|`WARNING`|30|警告信息，程序出现非预期情况，但不影响正常运行，如配置缺失使用默认值、接口响应超时重试|
|`ERROR`|40|错误信息，程序出现功能异常，某个业务流程执行失败，如数据库操作失败、接口调用异常|
|`CRITICAL`|50|严重错误，程序出现致命异常，无法继续运行，如数据库连接失败、核心配置缺失导致服务无法启动|

默认日志级别为`WARNING`，即仅会输出 WARNING 及以上级别的日志。

#### 2. 四大核心组件

logging 模块采用模块化设计，由四大核心组件协同工作，实现灵活的日志管理：

|组件|核心作用|
|---|---|
|`Logger`|日志器，程序直接使用的接口，负责创建、分发日志记录，每个模块通常使用独立的 Logger 实例|
|`Handler`|处理器，负责将日志记录输出到指定的渠道（控制台、文件、网络等），一个 Logger 可绑定多个 Handler|
|`Formatter`|格式化器，定义日志的输出格式，指定日志包含的字段、格式，一个 Handler 绑定一个 Formatter|
|`Filter`|过滤器，用于过滤日志记录，实现比日志级别更精细的日志控制，可绑定到 Logger 或 Handler|

### 8.4.3 logging 配置方式

#### 1. 基础快速配置：basicConfig

`logging.basicConfig()`是 logging 模块提供的快速配置方法，用于简单场景的日志配置，仅能配置根 Logger，必须在所有日志输出之前调用。

示例：

```python
import logging

# 基础配置
logging.basicConfig(
    # 日志级别
    level=logging.DEBUG,
    # 日志格式
    format="%(asctime)s - %(name)s - %(levelname)s - %(filename)s:%(lineno)d - %(message)s",
    # 日期格式
    datefmt="%Y-%m-%d %H:%M:%S",
    # 输出到文件，不指定则默认输出到控制台
    filename="app.log",
    # 文件写入模式，a为追加，w为覆盖
    filemode="a"
)

# 日志输出
logging.debug("这是DEBUG级别的调试信息")
logging.info("这是INFO级别的常规信息")
logging.warning("这是WARNING级别的警告信息")
logging.error("这是ERROR级别的错误信息")
logging.critical("这是CRITICAL级别的严重错误信息")

# 异常日志记录，自动记录完整异常栈
try:
    1 / 0
except ZeroDivisionError:
    logging.exception("除数为0，计算失败")
```

常用日志格式化字段：

|字段|描述|
|---|---|
|`%(asctime)s`|日志记录的时间，默认格式为`2024-05-01 12:00:00,000`|
|`%(name)s`|Logger 的名称|
|`%(levelname)s`|日志级别名称|
|`%(filename)s`|调用日志的文件名|
|`%(lineno)d`|调用日志的代码行号|
|`%(funcName)s`|调用日志的函数名|
|`%(message)s`|日志的具体内容|
|`%(process)d`|进程 ID|
|`%(thread)d`|线程 ID|

#### 2. 工程化标准配置：dictConfig 字典配置【推荐】

`logging.config.dictConfig()`是企业级项目的标准配置方式，通过字典定义完整的日志配置，支持多 Logger、多 Handler、多 Formatter 的复杂配置，灵活性极强，完全满足生产环境的需求。

工程化完整配置示例：

```python
import logging
import logging.config
import os
from datetime import datetime

# 日志存储目录
LOG_DIR = "logs"
if not os.path.exists(LOG_DIR):
    os.makedirs(LOG_DIR)

# 日志文件名称，按日期命名
LOG_FILE = os.path.join(LOG_DIR, f"app_{datetime.now().strftime('%Y-%m-%d')}.log")
ERROR_LOG_FILE = os.path.join(LOG_DIR, f"error_{datetime.now().strftime('%Y-%m-%d')}.log")

# 日志配置字典
LOGGING_CONFIG = {
    "version": 1,  # 配置格式版本，固定为1
    "disable_existing_loggers": False,  # 不禁用已存在的Logger
    # 格式化器定义
    "formatters": {
        # 详细格式，用于文件输出
        "verbose": {
            "format": "%(asctime)s - %(name)s - %(levelname)s - %(process)d - %(thread)d - %(filename)s:%(lineno)d - %(funcName)s - %(message)s",
            "datefmt": "%Y-%m-%d %H:%M:%S"
        },
        # 简洁格式，用于控制台输出
        "simple": {
            "format": "%(asctime)s - %(name)s - %(levelname)s - %(message)s",
            "datefmt": "%Y-%m-%d %H:%M:%S"
        }
    },
    # 过滤器定义（可选）
    "filters": {},
    # 处理器定义
    "handlers": {
        # 控制台处理器，输出INFO及以上级别日志
        "console": {
            "class": "logging.StreamHandler",
            "level": "INFO",
            "formatter": "simple",
            "stream": "ext://sys.stdout"
        },
        # 文件处理器，输出DEBUG及以上级别日志到通用日志文件
        "file": {
            "class": "logging.handlers.RotatingFileHandler",
            "level": "DEBUG",
            "formatter": "verbose",
            "filename": LOG_FILE,
            "maxBytes": 1024 * 1024 * 50,  # 单个文件最大50MB
            "backupCount": 10,  # 最多保留10个备份文件
            "encoding": "utf-8"
        },
        # 错误文件处理器，仅输出ERROR及以上级别日志到错误日志文件
        "error_file": {
            "class": "logging.handlers.TimedRotatingFileHandler",
            "level": "ERROR",
            "formatter": "verbose",
            "filename": ERROR_LOG_FILE,
            "when": "midnight",  # 每天午夜轮转
            "interval": 1,
            "backupCount": 30,  # 最多保留30天的日志
            "encoding": "utf-8"
        }
    },
    # 日志器定义
    "loggers": {
        # 项目根日志器
        "my_project": {
            "handlers": ["console", "file", "error_file"],
            "level": "DEBUG",
            "propagate": False  # 禁止向上传播到根日志器，避免重复日志
        },
        # 第三方库日志器，控制第三方库的日志输出
        "requests": {
            "handlers": ["console"],
            "level": "WARNING",
            "propagate": False
        }
    },
    # 根日志器，兜底配置
    "root": {
        "handlers": ["console"],
        "level": "WARNING"
    }
}

# 加载配置
logging.config.dictConfig(LOGGING_CONFIG)

# 模块内获取日志器，推荐使用__name__作为日志器名称，实现模块化日志管理
logger = logging.getLogger("my_project.user_service")

# 使用日志器
logger.debug("用户服务初始化完成")
logger.info("用户登录成功，user_id=1")
logger.warning("用户密码错误次数过多，user_id=1")
logger.error("用户信息查询失败，数据库连接超时")
logger.critical("用户服务启动失败，数据库无法连接")
```

核心配置说明：

- **日志轮转**：使用`RotatingFileHandler`按文件大小轮转，`TimedRotatingFileHandler`按时间轮转，避免单个日志文件过大，同时自动清理过期日志，是生产环境的强制要求；
- **模块化日志器**：每个模块使用`logging.getLogger(__name__)`获取独立的日志器，可针对不同模块设置不同的日志级别、处理器；
- **多 Handler 分离**：控制台输出简洁日志，文件输出详细日志，错误日志单独存储，便于线上问题排查；
- **编码设置**：文件处理器必须设置`encoding="utf-8"`，避免跨平台日志乱码。

### 8.4.4 日志管理最佳实践

1. **禁止使用 print () 输出业务日志**：所有业务日志、调试信息、异常信息必须使用 logging 模块输出，生产环境可统一管理；
2. **模块化日志器**：每个模块使用`logger = logging.getLogger(__name__)`创建独立的日志器，禁止直接使用根日志器`logging.xxx()`；
3. **合理设置日志级别**：开发环境使用`DEBUG`级别，生产环境使用`INFO`级别，避免大量 DEBUG 日志导致的性能损耗与磁盘占用；
4. **异常日志必须记录完整栈信息**：捕获异常时，必须使用`logger.exception()`记录异常，或在`logger.error()`中设置`exc_info=True`，保留完整的异常栈，便于问题排查；
5. **禁止日志中记录敏感信息**：密码、密钥、身份证号、手机号等敏感信息禁止写入日志，避免数据泄露；
6. **结构化日志**：生产环境推荐使用 JSON 格式的结构化日志，便于日志收集、检索、分析，可使用`python-json-logger`库实现；
7. **避免重复日志**：设置`propagate=False`，禁止日志向上传播，避免同一个日志被多个 Handler 重复输出；
8. **日志内容规范**：日志内容必须包含足够的上下文信息（如 user_id、order_id、request_id），便于问题追踪，禁止无意义的日志内容。

---
