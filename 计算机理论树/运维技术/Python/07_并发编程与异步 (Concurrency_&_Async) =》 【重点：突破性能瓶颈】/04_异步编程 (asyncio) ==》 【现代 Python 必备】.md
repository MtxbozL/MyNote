
异步编程是基于**协程（Coroutine）** 实现的单线程并发模型，是现代 Python 高并发 IO 密集型场景的首选方案，通过`asyncio`标准库提供完整的异步编程能力。其核心优势是：完全在用户态实现任务调度，无内核态线程切换的开销，单线程即可支持上万级的高并发，并发上限远高于多线程。

### 7.4.1 协程的核心概念

协程是**用户态的轻量级执行单元**，又称 “微线程”，其执行调度完全由程序自身控制，而非操作系统内核。协程运行在单线程内，通过`await`关键字主动让出 CPU 执行权，在 IO 等待时切换到其他协程执行，无需内核态上下文切换，开销极低。

协程与线程、进程的核心对比：

|特性|协程（asyncio）|多线程（threading）|多进程（multiprocessing）|
|---|---|---|---|
|调度主体|用户态，程序自身控制|内核态，操作系统调度|内核态，操作系统调度|
|上下文切换开销|极小，用户态切换，无内核开销|中等，内核态切换|极大，内核态完整进程切换|
|并发模型|单线程内并发，IO 多路复用|多线程并发，内核调度|多进程并行，多核利用|
|GIL 影响|单线程运行，无 GIL 竞争问题|IO 密集型影响小，CPU 密集型影响大|完全无影响，可利用多核|
|并发上限|极高，单线程支持上万级并发|中等，受线程数量限制，一般几百级|低，受 CPU 核心数限制，一般几十级|
|代码兼容性|必须使用异步库，同步代码会阻塞事件循环|兼容同步代码，无需修改原有逻辑|兼容同步代码，仅需处理进程间通信|
|适用场景|高并发 IO 密集型场景，如 Web 服务、爬虫、API 网关|中低并发 IO 密集型场景，简单同步代码并发|CPU 密集型场景，大数据计算|

### 7.4.2 异步编程核心语法

Python 3.5 + 引入`async/await`关键字（PEP 492 规范），是异步编程的核心语法，大幅简化了协程的编写与阅读。

#### 1. async def：定义协程函数

通过`async def`关键字定义的函数，称为**协程函数**，调用协程函数不会执行函数体代码，而是返回一个**协程对象**，必须放入事件循环中才能执行。

```python
import asyncio

# 定义协程函数
async def hello():
    print("Hello")
    # 模拟IO等待，主动让出执行权
    await asyncio.sleep(1)
    print("World")
    return "执行完成"

# 调用协程函数，返回协程对象，不会执行函数体
coro = hello()
print(type(coro))  # 输出：<class 'coroutine'>

# Python 3.7+ 启动协程的标准方式：asyncio.run()
# 自动创建事件循环，执行协程，关闭事件循环
result = asyncio.run(coro)
print(result)  # 输出：执行完成
```

#### 2. await：挂起协程，等待 IO 完成

`await`关键字用于挂起当前协程，等待**可等待对象（Awaitable）** 执行完成，期间主动让出 CPU 执行权，事件循环会调度其他协程执行，直到等待的 IO 完成后，再恢复当前协程的执行。

核心规则：

- `await`只能在`async def`定义的协程函数内部使用，不能在同步函数中使用；
- `await`后面必须跟随可等待对象，包括：协程对象、`Task`对象、`Future`对象、实现了`__await__`方法的异步对象；
- `await`是异步编程的核心，只有通过`await`才能实现协程的切换与并发。

### 7.4.3 事件循环（Event Loop）

事件循环是异步编程的**核心大脑**，是协程的调度器，负责：

1. 注册、管理、调度协程任务；
2. 监听 IO 事件，完成 IO 操作后唤醒对应的协程；
3. 处理异步回调、定时任务；
4. 协调同步代码与异步代码的执行。

Python 3.7 + 提供了极简的事件循环操作 API，无需手动创建与管理事件循环：

- `asyncio.run(coro)`：创建新的事件循环，执行传入的协程，协程完成后关闭事件循环，是启动异步程序的标准入口；
- `asyncio.get_running_loop()`：获取当前正在运行的事件循环对象，用于高级定制场景。

事件循环的核心执行模型：**单线程 + IO 多路复用**，同一时刻只有一个协程在执行，协程在 IO 等待时主动让出执行权，事件循环调度其他就绪的协程执行，实现单线程内的并发。

### 7.4.4 Task 任务：实现协程并发

直接使用`await`调用协程是**串行执行**的，只有前一个协程执行完成，才会执行后一个。要实现协程的并发执行，必须通过`asyncio.create_task()`将协程包装为`Task`任务，加入事件循环调度，实现并发执行。

`Task`是`Future`的子类，用于在事件循环中**并发调度协程**，创建后会立即加入事件循环的就绪队列，等待调度执行，无需手动启动。

#### 1. 串行 vs 并发对比

```python
import asyncio
import time

async def task(name: str, delay: int):
    print(f"任务 {name} 启动")
    await asyncio.sleep(delay)
    print(f"任务 {name} 完成")
    return delay

# 串行执行
async def serial_run():
    start = time.perf_counter()
    # 串行await，总耗时=1+2=3s
    await task("任务1", 1)
    await task("任务2", 2)
    end = time.perf_counter()
    print(f"串行执行总耗时：{end - start:.2f}s")

# 并发执行
async def concurrent_run():
    start = time.perf_counter()
    # 创建Task，立即加入事件循环调度
    task1 = asyncio.create_task(task("任务1", 1))
    task2 = asyncio.create_task(task("任务2", 2))
    # 等待两个任务完成，总耗时≈2s（最长任务的耗时）
    await task1
    await task2
    end = time.perf_counter()
    print(f"并发执行总耗时：{end - start:.2f}s")

# 启动程序
asyncio.run(serial_run())
asyncio.run(concurrent_run())
```

#### 2. asyncio.gather ()：批量并发执行

`asyncio.gather(*aws, return_exceptions=False)`是批量并发执行多个协程 / Task 的标准方法，等待所有任务完成后，按任务提交顺序返回结果列表。

```python
import asyncio
import time

async def task(name: str, delay: int):
    print(f"任务 {name} 启动")
    await asyncio.sleep(delay)
    print(f"任务 {name} 完成")
    return f"{name}结果"

async def main():
    start = time.perf_counter()
    # 批量并发执行多个协程
    results = await asyncio.gather(
        task("任务1", 1),
        task("任务2", 2),
        task("任务3", 1)
    )
    end = time.perf_counter()
    print(f"总耗时：{end - start:.2f}s")
    print(f"执行结果：{results}")  # 按提交顺序返回结果

asyncio.run(main())
```

核心参数`return_exceptions`：

- 默认`False`：任意一个任务抛出异常，会立即终止所有任务，向外抛出异常；
- 设置为`True`：任务抛出的异常会作为结果放入返回列表中，不会影响其他任务的执行，适用于需要保证所有任务都执行完成的场景。

#### 3. asyncio.as_completed ()：按完成顺序处理结果

`asyncio.as_completed(fs, timeout=None)`迭代传入的协程 / Task 列表，**哪个任务先完成，就先返回该任务的结果**，无需等待所有任务完成，适用于需要实时处理完成结果的场景。

### 7.4.5 异步生态

异步编程的核心限制是：**同步 IO 操作会阻塞整个事件循环**，导致所有协程暂停执行。因为同步 IO（如`requests.get()`、`time.sleep()`、同步文件读写）不会主动释放 GIL，也不会让出事件循环的执行权，会一直占用 CPU 直到 IO 完成，期间其他协程无法执行。

因此，异步编程必须使用**非阻塞的异步库**，替代传统的同步库，Python 拥有丰富的异步生态，覆盖全场景开发需求：

|场景|同步库|对应异步库|
|---|---|---|
|HTTP 请求|requests|aiohttp、httpx（同步 / 异步双支持）|
|数据库操作|pymysql、psycopg2|asyncpg（PostgreSQL）、aiomysql、aiosqlite|
|Redis 操作|redis-py|aioredis|
|MongoDB 操作|pymongo|motor|
|Web 框架|Flask、Django|FastAPI、Starlette、Sanic|
|消息队列|kafka-python|aiokafka|
|文件操作|内置 open ()|aiofiles|

示例：aiohttp 实现异步 HTTP 请求

```python
import asyncio
import aiohttp

async def fetch_url(session: aiohttp.ClientSession, url: str):
    async with session.get(url) as response:
        print(f"请求 {url}，状态码：{response.status}")
        return await response.text()

async def main():
    urls = [
        "https://www.python.org",
        "https://www.github.com",
        "https://www.baidu.com"
    ]
    # 异步HTTP会话，复用连接池
    async with aiohttp.ClientSession() as session:
        # 批量并发请求
        tasks = [fetch_url(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
        print(f"所有请求完成，共获取{len(results)}个页面")

asyncio.run(main())
```

### 7.4.6 异步编程最佳实践

1. **禁止在协程中执行同步 IO 与 CPU 密集型操作**：同步 IO 会阻塞事件循环，CPU 密集型操作会占用单线程，导致并发性能急剧下降；CPU 密集型操作需通过`ProcessPoolExecutor`交给多进程执行，同步 IO 可通过`ThreadPoolExecutor`交给多线程执行，通过`asyncio.run_in_executor()`集成到事件循环中。
2. **优先使用 asyncio.create_task () 实现并发**：避免直接串行 await，充分发挥异步的并发优势。
3. **使用 async with 管理异步上下文管理器**：异步文件、异步 HTTP 会话、异步数据库连接等资源，必须使用`async with`管理，保证资源的安全释放。
4. **异常处理必须精细化**：协程的异常不会自动打印，未捕获的异常会导致 Task 终止，必须在协程内部做好异常捕获，或通过`asyncio.gather(return_exceptions=True)`处理。
5. **避免长耗时的同步代码块**：协程中即使没有 IO，过长的同步计算也会阻塞事件循环，应拆分为多个小片段，通过`await asyncio.sleep(0)`主动让出执行权。

---

## 本章核心考点与学习要求

1. 深刻理解 GIL 全局解释器锁的底层本质、释放机制与存在原因，彻底掌握 GIL 对多线程的影响，明确 CPU 密集型与 IO 密集型任务的选型边界，能解释 Python 多线程 “伪多线程” 的核心原因。
2. 熟练掌握多线程的创建方式、核心 API 与生命周期管理，深刻理解线程安全问题的根源，熟练使用 Lock 互斥锁解决数据竞争问题，掌握死锁的产生条件与规避方案。
3. 熟练使用 ThreadPoolExecutor 线程池，掌握线程池的核心用法、参数调优与最佳实践，能基于线程池实现 IO 密集型任务的并发处理。
4. 熟练掌握多进程的创建方式、核心 API，深刻理解多进程与多线程的核心差异、适用场景，能基于多进程实现 CPU 密集型任务的多核并行计算。
5. 熟练掌握进程间通信的四种核心方案（Queue、Pipe、共享内存、Manager），明确各自的适用场景与性能差异，能实现安全的跨进程数据交互。
6. 熟练使用 ProcessPoolExecutor 进程池，掌握进程池的参数调优与最佳实践，能实现批量 CPU 密集型任务的高效调度。
7. 深刻理解协程的核心概念、异步编程的执行模型，熟练掌握`async/await`核心语法，能区分协程串行与并发的实现差异。
8. 掌握事件循环的核心作用、Task 任务的创建与调度，熟练使用`asyncio.gather()`实现批量协程的并发执行，掌握异步异常处理的最佳实践。
9. 理解异步编程的核心限制，熟悉 Python 主流异步生态库，能基于异步库实现高并发 IO 密集型业务场景。
10. 能根据业务场景的任务类型，精准选择多线程、多进程、异步 IO 三种并发方案，写出高性能、高稳定性的并发代码，解决相关的面试高频考点。