
线程是操作系统内核调度的**最小执行单元**，隶属于进程，同一个进程内的所有线程共享进程的内存地址空间、文件句柄等资源。Python 通过`threading`模块提供了完整的多线程编程能力，是 IO 密集型任务的轻量级并发方案。

### 7.2.1 线程的创建与核心 API

Python 提供两种线程创建方式，均基于`threading.Thread`类实现，遵循面向对象设计规范。

#### 1. 函数式创建（推荐，简单场景）

通过`Thread`类的`target`参数指定线程执行的函数，`args`/`kwargs`参数传递函数入参，是最常用的创建方式。

```python
import threading
import time

def task(name: str, delay: int):
    """线程执行的任务函数"""
    print(f"线程 {name} 启动，延迟 {delay}s 执行")
    time.sleep(delay)  # 模拟IO阻塞，会主动释放GIL
    print(f"线程 {name} 执行完成")

# 创建线程对象
t1 = threading.Thread(target=task, args=("线程1", 2))
t2 = threading.Thread(target=task, args=("线程2", 1))

# 启动线程，进入就绪状态，由操作系统调度执行
t1.start()
t2.start()

# 主线程等待子线程执行完成，阻塞主线程
t1.join()
t2.join()

print("所有线程执行完成")
```

#### 2. 类继承式创建（复杂场景）

继承`threading.Thread`类，重写`run()`方法，将线程执行逻辑封装在`run()`方法中，适用于逻辑复杂、需要封装状态的线程任务。

```python
import threading
import time

class CustomThread(threading.Thread):
    def __init__(self, name: str, delay: int):
        # 必须调用父类构造方法
        super().__init__(name=name)
        self.delay = delay

    # 重写run方法，线程启动后执行的核心逻辑
    def run(self):
        print(f"线程 {self.name} 启动，延迟 {self.delay}s 执行")
        time.sleep(self.delay)
        print(f"线程 {self.name} 执行完成")

# 创建线程实例
t1 = CustomThread("线程1", 2)
t2 = CustomThread("线程2", 1)

# 启动线程
t1.start()
t2.start()

# 等待线程完成
t1.join()
t2.join()
```

#### 3. 核心 API 详解

|方法 / 属性|功能描述|核心规则|
|---|---|---|
|`start()`|启动线程，将线程置为就绪状态|只能调用一次，多次调用会抛出`RuntimeError`，调用后操作系统会调度执行`run()`方法|
|`run()`|线程的核心执行逻辑|可被子类重写，`start()`会自动调用该方法，直接调用不会启动新线程，仅在当前线程同步执行|
|`join(timeout=None)`|阻塞当前线程，等待被调用线程执行完成|`timeout`为超时时间，单位秒，超时后不再阻塞；必须在`start()`之后调用|
|`is_alive()`|判断线程是否处于存活状态|线程启动后、执行完成前返回`True`，否则返回`False`|
|`daemon`|守护线程标记，布尔值|必须在`start()`之前设置；守护线程会在主线程退出时强制终止，无论是否执行完成；非守护线程（默认）会在主线程退出后继续执行，直到自身完成|
|`name`|线程名称|可自定义，用于调试与日志区分|
|`threading.current_thread()`|静态方法，返回当前线程对象|可在任务函数中获取当前线程的信息|
|`threading.active_count()`|静态方法，返回当前存活的线程总数|包含主线程|

### 7.2.2 线程安全与 Lock 互斥锁

#### 1. 线程安全问题的根源

同一个进程内的所有线程共享全局内存空间，多个线程同时修改同一个共享变量时，会出现**数据竞争（Race Condition）**，导致数据错乱。

需要明确的核心误区：**GIL 仅保证解释器内部数据的线程安全，不保护用户业务代码的线程安全**。因为用户的业务操作通常对应多条 Python 字节码，GIL 会在字节码之间释放，导致操作的非原子性。

示例：线程安全问题复现

```python
import threading

# 共享全局变量
count = 0
THREAD_NUM = 10
LOOP_NUM = 100000

def add_count():
    global count
    for _ in range(LOOP_NUM):
        # 该操作对应多条字节码，非原子性，会出现数据竞争
        count += 1

# 创建10个线程
threads = [threading.Thread(target=add_count) for _ in range(THREAD_NUM)]

# 启动所有线程
for t in threads:
    t.start()

# 等待所有线程完成
for t in threads:
    t.join()

# 预期结果：10*100000=1000000，实际结果远小于预期，出现数据错乱
print(f"最终计数：{count}")
```

#### 2. Lock 互斥锁解决方案

`threading.Lock`是 Python 提供的**互斥锁**，核心作用是将非原子性的业务操作包裹为临界区，保证同一时刻只有一个线程能执行临界区代码，彻底解决数据竞争问题。

锁的核心操作：

- `acquire(blocking=True, timeout=-1)`：获取锁，`blocking=True`时阻塞直到获取锁成功；`timeout`为超时时间，超时返回`False`。
- `release()`：释放锁，必须由持有锁的线程调用，释放后其他线程可竞争获取锁。

示例：使用 Lock 解决线程安全问题

```python
import threading

count = 0
THREAD_NUM = 10
LOOP_NUM = 100000
# 创建全局互斥锁
lock = threading.Lock()

def add_count():
    global count
    for _ in range(LOOP_NUM):
        # 进入临界区前获取锁
        lock.acquire()
        try:
            # 临界区代码，同一时刻只有一个线程执行
            count += 1
        finally:
            # 无论是否发生异常，都必须释放锁，避免死锁
            lock.release()

# 线程创建与启动逻辑同上
threads = [threading.Thread(target=add_count) for _ in range(THREAD_NUM)]
for t in threads:
    t.start()
for t in threads:
    t.join()

# 最终结果与预期完全一致：1000000
print(f"最终计数：{count}")
```

#### 3. 上下文管理器简化锁操作

`Lock`实现了上下文管理器协议，可通过`with`语句自动获取与释放锁，无需手动调用`acquire()`和`release()`，同时保证异常场景下锁的安全释放，是官方推荐的最佳实践。

```python
def add_count():
    global count
    for _ in range(LOOP_NUM):
        # with语句自动获取锁，代码块结束自动释放锁
        with lock:
            count += 1
```

### 7.2.3 死锁问题与规避方案

死锁是指两个或多个线程互相持有对方需要的锁，同时等待对方释放锁，导致所有线程永久阻塞，无法继续执行的问题。

#### 1. 死锁产生的四个必要条件

死锁必须同时满足以下四个条件，破坏其中任意一个即可避免死锁：

1. **互斥条件**：锁是排他性的，同一时刻只有一个线程能持有；
2. **持有并等待条件**：线程持有至少一个锁，同时申请其他被其他线程持有的锁，等待时不释放已持有的锁；
3. **不可剥夺条件**：线程已持有的锁，只能由自身主动释放，其他线程无法强制剥夺；
4. **循环等待条件**：多个线程形成循环等待链，每个线程等待下一个线程持有的锁。

#### 2. 死锁复现示例

```python
import threading

lock_a = threading.Lock()
lock_b = threading.Lock()

def thread1_func():
    with lock_a:
        print("线程1持有lock_a，等待lock_b")
        with lock_b:
            print("线程1持有lock_a和lock_b")

def thread2_func():
    with lock_b:
        print("线程2持有lock_b，等待lock_a")
        with lock_a:
            print("线程2持有lock_b和lock_a")

# 启动线程，大概率触发死锁，程序永久阻塞
t1 = threading.Thread(target=thread1_func)
t2 = threading.Thread(target=thread2_func)
t1.start()
t2.start()
```

#### 3. 死锁规避方案

1. **固定锁获取顺序**：所有线程按照统一的全局顺序获取锁，破坏循环等待条件，是最常用的规避方案。上述示例中，两个线程都先获取 lock_a，再获取 lock_b，即可避免死锁。
2. **设置锁超时时间**：`acquire()`设置`timeout`参数，超时后释放已持有的锁，破坏持有并等待条件。
3. **避免嵌套锁**：尽量减少锁的嵌套使用，降低死锁发生概率。
4. **一次性获取所有需要的锁**：线程要么一次性获取所有需要的锁，要么一个都不获取，破坏持有并等待条件。

### 7.2.4 ThreadPoolExecutor 线程池

线程池是基于`concurrent.futures`模块实现的线程管理工具，核心解决了手动创建线程的三大问题：① 线程频繁创建与销毁的性能开销；② 无限制创建线程导致的系统资源耗尽；③ 手动管理线程生命周期的复杂度。

线程池会预先创建固定数量的工作线程，任务提交后分配给空闲线程执行，任务执行完成后线程不销毁，继续等待新任务，同时通过最大线程数限制系统资源占用。

#### 1. 核心用法

```python
from concurrent.futures import ThreadPoolExecutor
import time

def task(name: str, delay: int):
    print(f"任务 {name} 启动，延迟 {delay}s")
    time.sleep(delay)
    print(f"任务 {name} 完成")
    return f"任务{name}结果"

# 方式1：with语句管理线程池，自动关闭
with ThreadPoolExecutor(max_workers=5) as executor:
    # 1. submit()：提交单个任务，返回Future对象
    future1 = executor.submit(task, "任务1", 2)
    future2 = executor.submit(task, "任务2", 1)

    # result()：阻塞等待任务完成，获取返回值
    print(future1.result())
    print(future2.result())

    # 2. map()：批量提交任务，按任务提交顺序返回结果
    tasks_args = [("任务3", 1), ("任务4", 2), ("任务5", 1)]
    results = executor.map(lambda x: task(*x), tasks_args)
    for res in results:
        print(res)

# 方式2：手动创建与关闭
executor = ThreadPoolExecutor(max_workers=5)
executor.submit(task, "任务6", 1)
executor.shutdown(wait=True)  # 等待所有任务完成后关闭线程池
```

#### 2. 核心 API 与参数

|参数 / 方法|功能描述|最佳实践|
|---|---|---|
|`max_workers`|线程池最大工作线程数|IO 密集型场景，推荐设置为`CPU核心数 * 5`；Python 3.8 + 默认值为`min(32, CPU核心数 + 4)`|
|`submit(fn, *args, **kwargs)`|提交单个任务，返回`Future`对象|适用于不同参数、不同函数的任务提交，可通过 Future 对象跟踪任务状态|
|`map(fn, *iterables)`|批量提交任务，按迭代顺序返回结果|适用于同一函数、不同参数的批量任务，结果顺序与任务提交顺序严格一致|
|`shutdown(wait=True)`|关闭线程池，不再接受新任务|`wait=True`时阻塞等待所有任务完成；`with`语句会自动调用该方法，无需手动处理|
|`Future.result(timeout=None)`|阻塞等待任务完成，获取返回值|`timeout`设置超时时间，超时抛出`TimeoutError`；任务执行异常时，调用该方法会抛出对应异常|
|`Future.done()`|判断任务是否完成，非阻塞|适用于轮询任务状态|
|`as_completed(fs)`|迭代 Future 对象列表，任务完成后立即返回|按任务完成顺序处理结果，无需等待所有任务完成，效率高于`map()`|

---
