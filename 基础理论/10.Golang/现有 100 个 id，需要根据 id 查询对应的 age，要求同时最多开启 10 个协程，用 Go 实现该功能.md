#### 核心标准答案

该需求的核心是**控制协程的最大并发数为 10**，同时完成 100 个 id 的查询任务，Go 语言中最常用、最优雅的实现方式有两种：**带缓冲 channel 的信号量模式**、**worker pool（协程池）模式**，其中信号量模式代码最简洁，适合该场景，worker pool 模式适合更复杂的批量任务场景。

##### 一、首选实现：带缓冲 channel 信号量模式（面试推荐）

核心原理：创建一个容量为 10 的带缓冲 channel，作为信号量，每个协程启动前先从 channel 中获取一个信号，任务完成后释放信号，保证同时运行的协程不超过 10 个；通过 sync.WaitGroup 等待所有任务完成，通过 sync.Map/ 切片收集结果，线程安全。

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// 模拟根据id查询age的接口
func getAgeById(id int) int {
    // 模拟接口耗时
    time.Sleep(10 * time.Millisecond)
    return id * 2 // 模拟返回结果
}

func main() {
    // 1. 准备100个id
    idCount := 100
    maxGoroutine := 10
    ids := make([]int, idCount)
    for i := 0; i < idCount; i++ {
        ids[i] = i + 1
    }

    // 2. 初始化组件
    var wg sync.WaitGroup
    sem := make(chan struct{}, maxGoroutine) // 信号量channel，容量10，控制最大并发
    resultMap := sync.Map{}                   // 线程安全的map，存储id-age结果

    // 3. 遍历所有id，提交任务
    for _, id := range ids {
        wg.Add(1)
        // 启动协程前，先获取信号量，缓冲满了会阻塞，控制并发数
        sem <- struct{}{}

        go func(id int) {
            // 任务完成后释放信号量和wg
            defer func() {
                <-sem
                wg.Done()
                // 捕获协程panic，避免单个任务失败导致整个程序崩溃
                if err := recover(); err != nil {
                    fmt.Printf("id=%d 查询失败，异常：%v\n", id, err)
                }
            }()

            // 执行查询
            age := getAgeById(id)
            resultMap.Store(id, age)
            fmt.Printf("id=%d 查询完成，age=%d\n", id, age)
        }(id)
    }

    // 4. 等待所有任务完成
    wg.Wait()
    close(sem) // 关闭channel

    // 5. 遍历结果，验证
    fmt.Println("\n所有任务完成，结果如下：")
    resultMap.Range(func(key, value interface{}) bool {
        fmt.Printf("id=%d, age=%d\n", key.(int), value.(int))
        return true
    })
}
```

##### 二、备选实现：Worker Pool（协程池）模式

核心原理：预先启动 10 个 worker 协程，作为固定的协程池，所有 id 任务通过 channel 分发给 worker，10 个 worker 并发处理任务，保证最大并发数为 10，适合大量任务的场景，资源占用更稳定。

go

运行

```
package main

import (
    "fmt"
    "sync"
    "time"
)

func getAgeById(id int) int {
    time.Sleep(10 * time.Millisecond)
    return id * 2
}

func main() {
    // 1. 初始化参数
    idCount := 100
    workerCount := 10
    ids := make([]int, idCount)
    for i := 0; i < idCount; i++ {
        ids[i] = i + 1
    }

    // 2. 创建任务channel和结果channel
    taskCh := make(chan int, idCount)
    resultCh := make(chan [2]int, idCount) // [id, age]
    var wg sync.WaitGroup

    // 3. 启动10个worker协程，固定并发数
    for i := 0; i < workerCount; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            // 从任务channel中消费id，直到channel关闭
            for id := range taskCh {
                func(id int) {
                    defer func() {
                        if err := recover(); err != nil {
                            fmt.Printf("id=%d 查询失败，异常：%v\n", id, err)
                        }
                    }()
                    age := getAgeById(id)
                    resultCh <- [2]int{id, age}
                }(id)
            }
        }()
    }

    // 4. 往任务channel中发送所有id
    go func() {
        for _, id := range ids {
            taskCh <- id
        }
        close(taskCh) // 所有任务发送完成，关闭channel，worker会自动退出
    }()

    // 5. 等待所有worker完成，关闭结果channel
    go func() {
        wg.Wait()
        close(resultCh)
    }()

    // 6. 收集结果
    resultMap := make(map[int]int)
    for res := range resultCh {
        id, age := res[0], res[1]
        resultMap[id] = age
    }

    // 7. 输出结果
    fmt.Println("所有任务完成，结果如下：")
    for id, age := range resultMap {
        fmt.Printf("id=%d, age=%d\n", id, age)
    }
}
```

#### 专家级拓展

- 两种实现的选型对比：
    
    表格
    
    |实现方式|优点|缺点|适用场景|
    |:--|:--|:--|:--|
    |信号量模式|代码简洁、灵活，任务提交和执行耦合度低，适合动态任务|短时间内创建大量协程（虽然同时运行的只有 10 个），有少量调度开销|任务数量不多（几百到几千）、快速实现的场景，面试首选|
    |Worker Pool 模式|固定协程数量，调度开销小，资源占用稳定，适合海量任务|代码相对复杂，需要管理任务和结果 channel|任务数量极大（上万到几十万）、长期运行的服务场景|
    
- 生产环境最佳实践：
    
    1. 必须在协程内加入 defer recover，捕获单个任务的 panic，避免单个任务的异常导致整个程序崩溃；
    2. 任务执行超时控制：生产环境中，必须给查询接口加上超时控制，避免单个任务阻塞协程，可通过 context.WithTimeout 实现；
    3. 错误重试：对于查询失败的任务，可加入重试机制，提升成功率；
    4. 限流控制：如果查询的是第三方接口，可加入限流逻辑，避免给下游服务造成过大压力。
    
- 进阶优化：带超时控制的实现，通过 context 控制整体任务的超时时间，避免任务无限阻塞：
    
    go
    
    运行
    
    ```
    // 主函数中加入context
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    // 任务执行时，监听ctx.Done()，超时直接退出
    ```
    

#### 面试避坑指南

- 严禁直接循环创建 100 个协程，不加任何并发控制，这是面试的核心扣分点，需求明确要求最多 10 个协程同时运行；
- 避免协程内循环变量的闭包陷阱，必须通过函数参数将 id 传入协程，否则所有协程都会拿到最后一个 id，这是 Go 面试高频错误；
- 不要遗漏 panic 捕获和结果的线程安全存储，sync.Map 是线程安全的，普通 map 在并发写入时会 panic，必须注意。