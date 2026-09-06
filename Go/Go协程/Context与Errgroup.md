# Context 与 Errgroup

## 1. Context 包

Context 是 Go 语言中用于传递截止时间、取消信号和请求范围值的标准库。在处理多 Goroutine 的级联取消时非常重要。

### 1.1 核心功能

- **取消信号传递**：当父级任务取消时，所有子 Goroutine 都能收到取消信号
- **截止时间控制**：可以设置超时时间，自动取消长时间运行的任务
- **请求范围值**：可以在请求链中传递元数据

### 1.2 常用函数

```go
// 创建带取消功能的 Context
ctx, cancel := context.WithCancel(context.Background())

// 创建带超时的 Context
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)

// 创建带截止时间的 Context
ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(5*time.Second))
```

### 1.3 使用范式

```go
func doTask(ctx context.Context) error {
    // 使用 select 监听取消信号
    select {
    case <-ctx.Done():
        // 收到取消信号
        return ctx.Err()
    case result := <-doWork():
        // 正常完成
        return nil
    }
}
```

## 2. Errgroup 包

Errgroup 是 Go 官方扩展同步库 `golang.org/x/sync/errgroup` 提供的工具，用于处理"一个失败，立刻取消其他所有"的场景。

### 2.1 核心功能

- **自动级联取消**：任何一个任务失败，自动取消所有其他任务
- **错误收集**：返回第一个发生的错误
- **统一等待**：等待所有任务完成

### 2.2 使用范式

```go
import "golang.org/x/sync/errgroup"

func processTasks() error {
    // 创建任务组和派生 Context
    g, ctx := errgroup.WithContext(context.Background())

    // 派发多个任务
    g.Go(func() error {
        return doTaskA(ctx)
    })
    g.Go(func() error {
        return doTaskB(ctx)
    })
    g.Go(func() error {
        return doTaskC(ctx)
    })

    // 等待所有任务完成
    return g.Wait()
}
```

### 2.3 级联取消的魔法

只要 `g.Go` 中有**任何一个**返回了 `error`，`errgroup` 就会**自动且立即**调用底层的 `cancel()` 函数，所有其他任务中的 `ctx.Done()` 管道都会被关闭，从而收到退出信号。

### 2.4 底层任务的编写范式

```go
func doAgentTask(ctx context.Context, taskName string) error {
    // 模拟一个长耗时操作的通道
    resultCh := make(chan string, 1)
    
    go func() {
        // 执行具体的检索逻辑
        resultCh <- "success"
    }()

    // 核心范式：使用 select 多路复用监听取消信号
    select {
    case <-ctx.Done():
        // 收到外部的取消信号
        fmt.Printf("任务 %s 被迫中断，释放资源\n", taskName)
        return ctx.Err()
        
    case res := <-resultCh:
        // 正常完成
        fmt.Printf("任务 %s 正常完成: %s\n", taskName, res)
        return nil
    }
}
```

## 3. 实际应用场景

### 3.1 OnCall 系统中的应用

在 OnCall 智能体系统中，当收到一个告警时，需要同时派发多个 Agent 任务：Agent A 查数据库、Agent B 查日志、Agent C 查监控。如果 Agent A 因为网络超时直接报错了，必须立刻停掉 B 和 C，避免白白消耗系统资源和 LLM 的 Token。

```go
func handleAlert(alert Alert) error {
    g, ctx := errgroup.WithContext(context.Background())

    g.Go(func() error {
        return queryDatabase(ctx, alert)
    })
    g.Go(func() error {
        return queryLogs(ctx, alert)
    })
    g.Go(func() error {
        return queryMetrics(ctx, alert)
    })

    return g.Wait()
}
```

### 3.2 并发网络请求

在处理大量并发网络请求时，使用 Context 可以有效控制超时和取消：

```go
func fetchURLs(urls []string) error {
    g, ctx := errgroup.WithContext(context.Background())
    
    for _, url := range urls {
        url := url // 捕获变量
        g.Go(func() error {
            req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
            resp, err := http.DefaultClient.Do(req)
            if err != nil {
                return err
            }
            defer resp.Body.Close()
            // 处理响应
            return nil
        })
    }
    
    return g.Wait()
}
```

## 4. 注意事项

1. **必须配合 Context 使用**：Errgroup 的取消功能依赖于 Context
2. **底层任务必须监听 ctx.Done()**：否则取消信号无法传递到具体任务
3. **错误处理**：`g.Wait()` 返回第一个发生的错误
4. **资源清理**：收到取消信号后要及时清理资源，避免泄漏
