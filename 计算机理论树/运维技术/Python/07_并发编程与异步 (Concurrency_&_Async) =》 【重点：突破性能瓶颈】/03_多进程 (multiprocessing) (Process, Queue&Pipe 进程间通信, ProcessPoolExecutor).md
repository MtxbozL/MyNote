
进程是操作系统**资源分配的最小单位**，每个进程拥有独立的内存地址空间、文件句柄、GIL 锁，完全不受其他进程的影响。Python 通过`multiprocessing`模块提供了与`threading`模块 API 高度一致的多进程编程能力，是 CPU 密集型任务的唯一有效并发方案。

### 7.3.1 多进程与多线程的核心对比

|特性|多进程（multiprocessing）|多线程（threading）|
|---|---|---|
|GIL 影响|每个进程拥有独立 GIL，可完全利用多核 CPU|同一进程内共享 GIL，CPU 密集型场景无法利用多核|
|内存空间|进程间内存完全独立，不共享数据|线程间共享进程内存空间，数据共享便捷|
|上下文切换开销|大，操作系统需切换完整的进程内存、上下文|小，仅需切换线程的栈与寄存器|
|创建与销毁开销|大，进程创建需要操作系统分配独立资源|小，轻量级，创建速度快|
|通信方式|需通过 IPC 机制（Queue、Pipe、共享内存等）实现，复杂度高|可直接通过共享变量 + 锁实现，简单便捷|
|稳定性|高，子进程崩溃不影响主进程与其他进程|低，子线程崩溃会导致整个进程崩溃|
|适用场景|CPU 密集型计算、大数据处理、需要高稳定性的任务|IO 密集型任务、轻量级并发、需要频繁数据共享的场景|

### 7.3.2 进程的创建与核心 API

与线程一致，进程也提供两种创建方式，基于`multiprocessing.Process`类实现，API 与`threading.Thread`高度一致，降低了学习成本。

**核心注意事项**：Windows 系统下，多进程代码必须放在`if __name__ == '__main__':`代码块中，否则会导致子进程无限递归创建，触发运行时错误。

#### 1. 函数式创建

```python
from multiprocessing import Process
import time
import os

def task(name: str, delay: int):
    """进程执行的任务函数"""
    print(f"进程 {name} 启动，PID: {os.getpid()}，父进程PPID: {os.getppid()}")
    time.sleep(delay)
    print(f"进程 {name} 执行完成")

# Windows系统必须放在main代码块中
if __name__ == '__main__':
    # 创建进程对象
    p1 = Process(target=task, args=("进程1", 2))
    p2 = Process(target=task, args=("进程2", 1))

    # 启动进程
    p1.start()
    p2.start()

    # 等待进程执行完成
    p1.join()
    p2.join()

    print(f"主进程执行完成，PID: {os.getpid()}")
```

#### 2. 类继承式创建

```python
from multiprocessing import Process
import time
import os

class CustomProcess(Process):
    def __init__(self, name: str, delay: int):
        super().__init__(name=name)
        self.delay = delay

    def run(self):
        print(f"进程 {self.name} 启动，PID: {os.getpid()}")
        time.sleep(self.delay)
        print(f"进程 {self.name} 执行完成")

if __name__ == '__main__':
    p1 = CustomProcess("进程1", 2)
    p2 = CustomProcess("进程2", 1)
    p1.start()
    p2.start()
    p1.join()
    p2.join()
    print("主进程执行完成")
```

#### 3. 核心 API 详解

`Process`类的核心 API 与`Thread`类完全一致，核心差异点如下：

- `pid`属性：返回进程的 PID（操作系统进程 ID），唯一标识进程；
- `terminate()`：强制终止进程，立即停止进程执行，不会执行清理逻辑，可能导致资源泄漏，谨慎使用；
- `kill()`：与`terminate()`功能一致，Unix 系统下发送 SIGKILL 信号，Windows 下与`terminate()`等价；
- `daemon`：守护进程标记，守护进程会在主进程退出时立即终止，无法创建子进程。

### 7.3.3 进程间通信（IPC）

进程间内存完全隔离，无法直接共享变量，必须通过操作系统提供的 IPC（Inter-Process Communication，进程间通信）机制实现数据交互，`multiprocessing`模块提供了多种 IPC 方案，适配不同场景。

#### 1. Queue：消息队列（最常用）

`multiprocessing.Queue`是基于管道 + 锁实现的**进程安全、线程安全**的消息队列，支持多生产者、多消费者模型，是进程间通信的首选方案，适用于批量数据传递、任务分发场景。

核心 API：

- `put(obj, block=True, timeout=None)`：向队列中放入数据，队列满时阻塞；
- `get(block=True, timeout=None)`：从队列中取出数据，队列空时阻塞；
- `empty()`：判断队列是否为空；
- `full()`：判断队列是否已满；
- `qsize()`：返回队列中当前元素数量。

示例：

```python
from multiprocessing import Process, Queue
import time

def producer(q: Queue):
    """生产者进程：向队列中放入数据"""
    for i in range(5):
        data = f"数据{i}"
        q.put(data)
        print(f"生产者放入：{data}")
        time.sleep(0.5)

def consumer(q: Queue):
    """消费者进程：从队列中取出数据"""
    while True:
        data = q.get()
        if data == "END":
            break
        print(f"消费者取出：{data}")
        time.sleep(1)

if __name__ == '__main__':
    # 创建消息队列
    q = Queue(maxsize=10)
    # 创建生产者与消费者进程
    p_producer = Process(target=producer, args=(q,))
    p_consumer = Process(target=consumer, args=(q,))

    p_producer.start()
    p_consumer.start()

    # 等待生产者完成
    p_producer.join()
    # 放入结束标记
    q.put("END")
    # 等待消费者完成
    p_consumer.join()
```

#### 2. Pipe：管道

`multiprocessing.Pipe`是双向的管道通信机制，创建时返回一对连接对象`(conn1, conn2)`，分别代表管道的两端，一端发送数据，另一端接收数据，适用于两个进程之间的点对点通信，性能高于 Queue。

核心 API：

- `send(obj)`：向管道另一端发送数据；
- `recv()`：从管道中接收数据，无数据时阻塞；
- `close()`：关闭管道连接。

示例：

```python
from multiprocessing import Process, Pipe
import time

def worker(conn):
    """子进程：接收主进程数据，处理后返回"""
    while True:
        data = conn.recv()
        if data == "END":
            break
        print(f"子进程收到：{data}")
        conn.send(f"处理完成：{data}")
    conn.close()

if __name__ == '__main__':
    # 创建管道，duplex=True为双向管道，False为单向管道
    parent_conn, child_conn = Pipe(duplex=True)
    p = Process(target=worker, args=(child_conn,))
    p.start()

    # 主进程发送数据
    for i in range(3):
        parent_conn.send(f"任务{i}")
        res = parent_conn.recv()
        print(f"主进程收到：{res}")
        time.sleep(0.5)

    parent_conn.send("END")
    p.join()
```

#### 3. 共享内存：Value/Array

`multiprocessing.Value`和`multiprocessing.Array`是基于操作系统共享内存实现的 IPC 方案，直接在进程间共享内存块，无需序列化 / 反序列化，**性能最高**，适用于大量数值数据的高速共享场景。

核心规则：

- `Value(typecode, value)`：共享单个数值，`typecode`为类型码，如`'i'`代表 int，`'d'`代表 double；
- `Array(typecode, size_or_initializer)`：共享固定长度的数组，元素类型统一；
- 共享内存的操作不是进程安全的，必须配合`Lock`互斥锁保证数据安全。

示例：

```python
from multiprocessing import Process, Value, Array, Lock

def increment_count(count: Value, lock: Lock):
    for _ in range(100000):
        with lock:
            count.value += 1

def modify_array(arr: Array, lock: Lock):
    with lock:
        for i in range(len(arr)):
            arr[i] *= 2

if __name__ == '__main__':
    # 创建共享内存与锁
    lock = Lock()
    shared_count = Value('i', 0)  # 共享int类型数值，初始值0
    shared_array = Array('i', [1, 2, 3, 4, 5])  # 共享int数组

    # 创建进程
    p1 = Process(target=increment_count, args=(shared_count, lock))
    p2 = Process(target=increment_count, args=(shared_count, lock))
    p3 = Process(target=modify_array, args=(shared_array, lock))

    p1.start()
    p2.start()
    p3.start()

    p1.join()
    p2.join()
    p3.join()

    print(f"共享计数结果：{shared_count.value}")  # 输出：200000
    print(f"共享数组结果：{shared_array[:]}")  # 输出：[2, 4, 6, 8, 10]
```

#### 4. Manager：管理器

`multiprocessing.Manager`提供了跨进程共享的高级数据类型，如`list`、`dict`、`Lock`、`Namespace`等，支持网络间进程通信，使用简单，无需手动处理锁机制，适用于复杂数据结构的跨进程共享，性能低于共享内存与 Queue。

示例：

```python
from multiprocessing import Process, Manager

def modify_dict(shared_dict):
    shared_dict["name"] = "Python"
    shared_dict["version"] = 3.12

def modify_list(shared_list):
    shared_list.append(1)
    shared_list.append(2)

if __name__ == '__main__':
    # 创建管理器
    with Manager() as manager:
        # 创建共享字典与列表
        shared_dict = manager.dict()
        shared_list = manager.list()

        # 创建进程
        p1 = Process(target=modify_dict, args=(shared_dict,))
        p2 = Process(target=modify_list, args=(shared_list,))

        p1.start()
        p2.start()
        p1.join()
        p2.join()

        print(f"共享字典：{shared_dict}")  # 输出：{'name': 'Python', 'version': 3.12}
        print(f"共享列表：{shared_list}")  # 输出：[1, 2]
```

### 7.3.4 ProcessPoolExecutor 进程池

与线程池一致，进程池基于`concurrent.futures.ProcessPoolExecutor`实现，解决了手动创建进程的开销大、资源占用不可控的问题，API 与`ThreadPoolExecutor`完全一致，可无缝切换。

核心特性：

- `max_workers`默认值为 CPU 核心数，CPU 密集型场景下，推荐设置为与 CPU 核心数一致，避免过多进程导致的上下文切换开销；
- 任务与返回值必须支持 pickle 序列化，因为进程间数据传递需要序列化；
- Windows 系统下，任务函数必须定义在顶层作用域，不能是嵌套函数或 lambda 表达式。

示例：

```python
from concurrent.futures import ProcessPoolExecutor
import math

def is_prime(n: int) -> bool:
    """CPU密集型任务：判断是否为质数"""
    if n < 2:
        return False
    for i in range(2, int(math.isqrt(n)) + 1):
        if n % i == 0:
            return False
    return True

if __name__ == '__main__':
    numbers = range(100000, 101000)
    # 创建进程池，max_workers为CPU核心数
    with ProcessPoolExecutor() as executor:
        # 批量提交任务
        results = executor.map(is_prime, numbers)
    
    # 统计质数数量
    prime_count = sum(results)
    print(f"100000-101000之间的质数数量：{prime_count}")
```

---
