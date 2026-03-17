#### 核心标准答案

单个 Goroutine 出现 panic，核心影响和处理规则如下：

1. **核心执行流程**
    
    - 当某个 Goroutine 内出现 panic 时，会立即终止当前 Goroutine 的正常代码执行，开始执行该 Goroutine 内注册的 defer 函数，按 defer 的逆序执行；
    - 如果 defer 函数中没有使用`recover()`捕获 panic，panic 会沿着调用栈向上传播，直到该 Goroutine 的所有 defer 执行完成，最终会导致整个 Go 程序崩溃退出，所有 Goroutine 都会被终止；
    - 如果 defer 函数中通过`recover()`成功捕获了 panic，panic 会被终止，程序不会崩溃，该 Goroutine 会完成剩余的 defer 函数执行，之后正常退出，不会影响其他 Goroutine 的正常运行。
    
2. **核心特性**
    
    - panic 的影响范围是**当前 Goroutine**，只有当前 Goroutine 的 defer 函数能捕获到它，其他 Goroutine 的 defer/recover 无法捕获跨协程的 panic；
    - 即使父 Goroutine 创建了子 Goroutine，子 Goroutine 的 panic 也无法被父 Goroutine 的 recover 捕获，二者的 panic 是完全隔离的；
    - panic 只能在 defer 函数中被 recover 捕获，非 defer 函数中的 recover 是无效的，无法捕获 panic。
    

#### 专家级拓展

- 正确的 panic 处理代码示例（面试可直接写出）：
    
    go
    
    运行
    
    ```
    package main
    
    import (
        "fmt"
        "time"
    )
    
    func main() {
        // 主协程的recover无法捕获子协程的panic
        defer func() {
            if err := recover(); err != nil {
                fmt.Println("主协程捕获到panic：", err)
            }
        }()
    
        // 启动子协程，必须在子协程内部注册defer recover
        go func() {
            // 子协程内部必须注册defer recover，才能捕获自身的panic
            defer func() {
                if err := recover(); err != nil {
                    fmt.Println("子协程捕获到panic：", err)
                    // 可在这里记录panic日志、堆栈信息，做告警、降级处理
                }
            }()
    
            // 模拟panic场景
            fmt.Println("子协程开始执行")
            panic("子协程出现异常")
            // panic之后的代码不会执行
            fmt.Println("子协程执行结束")
        }()
    
        // 主协程继续运行，不受子协程panic的影响（子协程recover成功）
        time.Sleep(1 * time.Second)
        fmt.Println("主协程正常执行结束")
    }
    ```
    
- 生产环境最佳实践：
    
    1. **所有启动的 Goroutine，必须在入口处注册 defer recover**，尤其是业务服务中异步启动的协程，避免单个协程的 panic 导致整个服务崩溃，这是 Go 后端开发的铁律；
    2. recover 捕获 panic 后，必须记录完整的堆栈信息，而不是只打印错误信息，可通过`debug.Stack()`获取 panic 的堆栈，方便定位问题；
    3. 对于核心业务的协程，panic 捕获后，可做降级处理、重试机制，而不是直接让协程退出，保证业务可用性；
    4. 不要滥用 recover，对于无法恢复的严重 panic（比如内存不足），即使捕获了，也应该让程序优雅退出，避免出现未知的异常。
    
- 特殊场景：panic 在 defer 中再次触发，只有最后一次 panic 能被 recover 捕获；如果 defer 中嵌套 panic，会覆盖之前的 panic，最终程序崩溃。

#### 面试避坑指南

- 严禁说「父协程的 recover 可以捕获子协程的 panic」，这是 Go 面试最高频的错误，panic 是协程隔离的，跨协程无法捕获，会直接被判定为对 Go 的 panic 机制理解错误；
- 避免说「panic 只会终止当前协程，不会影响整个程序」，只有 recover 成功捕获了 panic，才不会影响其他协程；如果没有 recover，最终会导致整个程序崩溃，这是核心区别，必须说清楚；
- 不要遗漏「recover 必须在 defer 函数中才能生效」这个核心规则，这是面试必问的细节点。