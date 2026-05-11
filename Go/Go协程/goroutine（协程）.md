---
title: "4.3 学习 Go 协程：goroutine | Go 编程时光"
source: "https://go.sixue.work/golang/c04/c04_03.html"
author:
published: 2026-04-21
created: 2026-05-10
description: "基于 VitePress 的 Go 语言完整学习网站"
tags:
  - "clippings"
---
##  学习 Go 协程：goroutine
[[Golang面试题#1.2 什么是协程？ Go协程]]
说到 Go 语言，许多人对它的第一印象就是从语言层面原生支持并发，这使得开发者可以非常方便地编写高性能且易于理解的程序。

在 Python 等主流编程语言中，并发编程的门槛较高。你需要学习多进程、多线程的概念，还要熟悉各种支持并发的库，比如 asyncio、aiohttp 等，同时要了解它们之间的区别和优缺点，能够根据场景选择合适的并发模式。

而 Go 作为一门现代化的编程语言，这些复杂的问题已经被大大简化。在 Go 中，开发者无需关心进程池/线程池的创建，也无需考虑多线程和多进程的具体应用场景。它原生提供了非常优秀的 **goroutine** （协程）机制，能够自动高效地处理并发任务，你只需简单地调用即可。

## 一、并发 (Concurrency) 与并行 (Parallelism)

在计算机科学中， **并发** 和 **并行** 是两个经常被混淆的概念。它们虽然都涉及到同时处理多个任务，但有着本质的区别：

#### 并发

并发是指在同一时间段内，多个任务看似同时执行，但实际上是通过快速切换轮流执行的。在单核 CPU 系统中，由于只有一个处理器核心，同一时刻只能执行一个任务。然而，操作系统通过时间片轮转调度算法，在不同任务之间快速切换，使得用户感觉多个任务好像是同时在运行。

- **本质：** 是指 **处理** 很多事情的能力。
- **特点：** 在同一个 **时间段** 内，多个任务都在向前推进。
- **底层：** 在单核 CPU 上，通过快速地切换任务（时间片轮转），给人一种“同时进行”的错觉。它关注的是 **任务结构的组织** 。

#### 并行

并行指的是在同一时刻，多个任务在不同的处理器核心或者不同的计算资源上真正地同时执行。这要求系统具备多个物理处理单元，如多核 CPU 或多台计算机组成的集群。

- **本质：** 是指 **同时做** 很多事情的能力。
- **特点：** 在同一个 **时刻** ，多个任务真正地同时执行。
- **底层：** 必须依赖硬件支持（如多核 CPU 或分布式集群）。它关注的是 **物理上的同时执行** 。

并发是 “逻辑上的同时”（任务切换），并行是 “物理上的同时”（多核执行）。

## 一、Goroutine 是什么？

在 Go 语言中，goroutine 是实现并发编程的核心概念，它是一种轻量级的线程。与操作系统原生线程相比，goroutine 的创建和销毁开销极小，使得 Go 语言能够轻松管理大量的并发任务。

### 1\. 轻量级线程

goroutine 是由 Go 运行时（runtime）管理的轻量级执行单元。与操作系统线程（OS 线程）相比，它的资源占用非常少，一个程序可以轻松创建数以万计的 goroutine。例如，创建一个 OS 线程可能需要1MB 或更大的空间，而一个新的 goroutine 初始时仅需要 2KB 左右的栈空间，并且这个栈空间可以根据需要动态增长和收缩。

**总结：** Goroutine 相比传统线程更加轻量、高效，使得 Go 语言能以极低的成本实现大规模并发。

### 2\. 并发执行

多个 goroutine 可以并发执行，Go 运行时的调度器负责将这些 goroutine 合理地分配到操作系统线程上，从而实现并发处理。在单核 CPU 上，多个 goroutine 会在同一时间片内交替执行，表现为并发；而在多核 CPU 上，多个 goroutine 可以同时在不同的核心上并行执行。需要注意的是，并发并不等同于并行。

Go 语言同时支持并发和并行，其独特的设计让两者的实现变得简单高效。

### 3.Goroutine vs 操作系统线程

| 特性 | Goroutine (协程) | Thread (操作系统线程) |
| --- | --- | --- |
| **内存占用** | **极小** ，初始栈空间仅 几KB (通常 2KB)，可动态伸缩。 | **较大** ，初始栈空间通常为 1MB 或更大。 |
| **创建开销** | **极低** ，只需要很少的 CPU 时间。 | **较高** ，涉及到操作系统内核的调度。 |
| **调度** | 由 Go 语言的 **运行时（Runtime）** 负责调度（M:N 调度模型）。 | 由操作系统内核负责调度。 |
| **数量** | 可轻松创建 **数十万个** Goroutine。 | 数量有限，创建过多会耗尽系统资源。 |

## 二、如何启动 Goroutine

启动 Goroutine 非常简单，只需在一个函数调用前加上 `go` 关键字。

```go
// 执行一个函数
func()

// 开启一个协程执行这个函数
go func()
```
### 1\. 主协程与子协程

在 Go 程序中，main 函数是程序的入口，main 函数所运行的协程被称为 main Goroutine（ **主协程** ）。

<font color="#4bacc6">只有在 main 函数及其调用的函数代码内</font>，才能够使用 go + func() 的方式启动新的协程，这些新启动的协程被称作 **子协程** 。

main 函数所处的主协程类似于传统编程模型中的主线程， **一旦 main 函数执行完毕，主协程随即结束** ，此时，无论正在运行的子协程代码是否执行完成，都会被强制终止。

**示例：**

```go
package main

import "fmt"

func mytest() {
    fmt.Println("hello, go")
}

func main() {
    // 启动一个协程
    go mytest()
    fmt.Println("hello, world")
}
```

在上述代码中，go mytest()语句启动了一个新的 goroutine 来执行mytest函数。main函数会继续执行后续代码，不会等待mytest函数执行完毕。

为了让大家看到效果，我们可以临时使用 time.Sleep 来阻塞 main 函数（虽然这在实际开发中不推荐，但适合演示）：

```go
package main

import (
    "fmt"
    "time"
)

func mytest() {
    fmt.Println("hello, go")
}

func main() {
    go mytest()
    fmt.Println("hello, world")
    time.Sleep(time.Second) // 等待 1 秒
}
```

输出如下：

```
hello, world
hello, go
```

### 2\. 多个协程并发执行的效果

为了让你看到并发的效果，这里举个最简单的例子：

```go
package main

import (
    "fmt"
    "time"
)

func mygo(name string) {
    for i := 0; i < 5; i++ {
        fmt.Printf("In goroutine %s\n", name)
        // 为了避免第一个协程执行过快，观察不到并发的效果，加个休眠
        time.Sleep(10 * time.Millisecond) 
    }
}

func main() {
    go mygo("协程1号") // 第一个协程
    go mygo("协程2号") // 第二个协程
    
    // 同样需要等待子协程执行完成
    time.Sleep(time.Second)
}
```

主协程和子协程并发执行，它们的执行顺序是不确定的。这是因为 Go 语言的调度器会在多个可运行的协程之间进行切换，而调度的时机和策略是由 Go 运行时决定的。

输出如下（顺序可能不同），可以观察到两个协程交替执行：

```
In goroutine 协程2号
In goroutine 协程1号
In goroutine 协程1号
In goroutine 协程2号
In goroutine 协程2号
In goroutine 协程1号
...
```

## 三、Goroutine 的生命周期

Goroutine 的生命周期包含几个关键阶段：创建、运行、阻塞、唤醒以及结束：

### 1\. 创建

```go
func task() {
    // 任务逻辑
    fmt.Println("Goroutine is running")
}

func main() {
    go task()
    // 主协程的其他逻辑
}
```

在上述代码中， `go task()` 语句创建并启动了一个新的Goroutine来执行 `task` 函数。此时，新的Goroutine被创建并进入可运行状态，等待Go运行时调度器将其分配到一个操作系统线程上执行。

### 2\. 运行

当调度器将Goroutine分配到一个操作系统线程上时，Goroutine开始运行其关联函数的代码。在运行过程中，Goroutine会顺序执行函数中的语句，直到遇到阻塞操作、函数返回或者发生异常。例如：

```go
func runningGoroutine() {
    for i := 0; i < 5; i++ {
        fmt.Printf("Goroutine is running: %d\n", i)
        time.Sleep(100 * time.Millisecond)
    }
}
```

在这个 `runningGoroutine` 函数中，Goroutine会循环打印数字，并每次暂停100毫秒，在此期间，Goroutine处于运行状态。

### 3\. 阻塞

Goroutine在运行过程中可能会遇到一些操作导致其进入阻塞状态，暂停执行，让出CPU资源给其他可运行的Goroutine。常见的阻塞操作包括：

- **I/O操作** ：如文件读写、网络请求等。例如，当Goroutine执行 `io.Read` 或 `http.Get` 等操作时，它会被阻塞，直到I/O操作完成。
```go
func ioBlocking() {
    file, err := os.Open("example.txt")
    if err != nil {
        log.Fatal(err)
    }
    defer file.Close()

    buffer := make([]byte, 1024)
    _, err = file.Read(buffer)
    if err != nil && err != io.EOF {
        log.Fatal(err)
    }
    // 文件读取完成后继续执行
}
```
- **通道操作** ：当Goroutine尝试从一个空的无缓冲通道[[Channel（通道）]]接收数据，或者向一个已满的无缓冲通道发送数据时，会发生阻塞，直到有其他Goroutine向该通道发送或接收数据。
```go
func channelBlocking() {
    ch := make(chan int)
    value := <-ch // 阻塞在这里，直到有数据发送到ch
    fmt.Println("Received:", value)
}
```
- **同步原语操作** ：例如使用 `sync.Mutex` 进行加锁操作时，如果锁已经被其他Goroutine持有，当前Goroutine会被阻塞，直到锁被释放。
```go
func mutexBlocking() {
    var mu sync.Mutex
    mu.Lock() // 阻塞在这里，如果锁已被占用
    // 临界区代码
    mu.Unlock()
}
```

### 4\. 唤醒

处于阻塞状态的Goroutine在相关条件满足时会被唤醒，重新进入可运行状态，等待调度器再次分配CPU资源继续执行。例如：

- **I/O操作完成** ：当文件读取或网络请求完成时，对应的Goroutine会被唤醒。
- **通道操作满足条件** ：如果有数据发送到一个正在等待接收的通道，或者有Goroutine从一个正在等待发送的通道接收数据，相关的Goroutine会被唤醒。
- **同步原语释放** ：当持有锁的Goroutine释放了 `sync.Mutex` 锁，等待该锁的Goroutine会被唤醒。

### 5\. 结束

Goroutine的生命周期在以下几种情况下结束：

- **主协程结束：** 主Goroutine的结束意味着整个程序的结束，无论此时子Goroutine是否执行完成。
- **函数正常返回** ：Goroutine协程会一直运行，直到函数中的所有语句都执行完毕并返回。
- **发生未捕获的异常** ：如果Goroutine在执行过程中发生了 [未被recover捕获的panic](https://go.sixue.work/golang/c01/c01_15.html#%E4%B8%89%E3%80%81-panic-%E6%97%A0%E6%B3%95%E8%B7%A8%E8%B6%8A-goroutine-%E8%BE%B9%E7%95%8C) ，Goroutine会终止，并导致程序崩溃

理解Goroutine的生命周期对于编写高效、健壮的并发程序至关重要，避免出现死锁、资源泄漏等问题。

## 四、Go 调度器原理
[[Golang面试题#9 . GMP面试题]]
Go 语言的调度器采用 **GMP 模型** ，这是 Go 能够实现高性能并发的核心：
- **G (Goroutine)**: 协程，用户态的轻量级线程，包含栈、指令指针等信息。
- **M (Machine)**: 操作系统线程，真正执行计算的资源，由操作系统调度。
- **P (Processor)**: 处理器（逻辑处理器），调度的上下文，维护了一个 Goroutine 队列。M 必须持有 P 才能执行 G。
Go 运行时会维护 `GOMAXPROCS` 个 P，通常默认为 CPU 核心数。
1. **复用线程** ：避免频繁创建、销毁线程，而是复用线程。
2. **利用多核** ：通过 M 和 P 的绑定，让多个线程同时工作。
3. **抢占式调度** ：防止某个 Goroutine 长时间占用 CPU，导致其他 Goroutine 饿死。
4. **全局队列与本地队列** ：P 维护本地队列，还有一个全局队列，平衡负载。
### 一、 GMP 核心角色定义
- **G (Goroutine)**：分布式调度单位。
    - 它是用户态的“轻量级线程”，初始栈仅 2KB。它包含栈信息、状态及要执行的任务函数。
- **M (Machine)**：内核线程的封装。
    - 它是真正的执行单元，所有的代码最终都要在 M 上运行。M 必须绑定一个 P 才能执行 G。
- **P (Processor)**：逻辑处理器/上下文。
    - 它是 G 和 M 之间的“中间层”。P 维护着一个本地运行队列（LRQ），并决定了系统的并行度。
![[Pasted image 20260510154708.png]]
---
### 二、 动态调度全流程

> [!NOTE]
> 面试官通常会问：“一个 G 是如何被执行的？”你可以按照以下逻辑回答：
#### 1. 生产与存储阶段
- 当通过 `go func()` 创建 G 时，它会优先放入当前 P 的 **本地队列（LRQ）**。
- 如果本地队列已满（256 个），该 G 及其余部分 G 会被移至 **全局队列（GRQ）** 中。
#### 2. M 的消费与执行阶段

M 为了执行 G，会按照以下优先级进行搜索：

1. **本地队列**：直接从绑定的 P 队列中获取。
    
2. **全局队列**：如果本地为空，去全局队列批量拉取（需加锁）。
    
3. **工作窃取**：如果全局也为空，M 会尝试从其他繁忙的 P 那里“偷”走一半的 G 来执行。   

	M寻找G的顺序：**本地队列(LRQ) -> 全局队列(GRQ) -> 工作窃取(Work-Stealing)**。
---
### 三、 核心设计机制
![[Pasted image 20260510154737.png]]

面试的深度往往体现在对这几个机制的理解上：
#### 1. 工作窃取 (Work-Stealing) —— 解决负载不均（图中左侧）
当某个 M 闲置而其他 M 忙碌时，空闲的 M 会通过绑定的 P 去偷取其他 P 队列中的 G。
- **场景**：图中 P2 对应的本地队列空了，而 P1 还有很多 G。
	
- **动作**：为了不让 M2 闲着，P2 会从 P1 的队列尾部**偷走一半**的 G。
	
- **意义**：保证了所有 CPU 核心都能被充分利用，避免了“一核有难，多核围观”的尴尬。
#### 2. 移交机制 (Hand Off) —— 解决系统调用阻塞（图右上角）

- 当 M1 执行的 G1 发生 **Syscall（阻塞式系统调用）** 时，M1 会与 P 脱离并陷入阻塞
    
- P 会寻找或创建一个新的 M2 来接管剩下的本地队列，保证后续 G 不被卡死。
    
- 当 G1 调用结束，M1 会尝试找回 P，若找不回则将 G1 放入全局队列并进入休眠池。
#### 3. 抢占式调度 (Preemption) —— 解决公平性（图右下角）

- Go 运行时有一个监控线程（sysmon），会定期检查运行超过 **10ms** 的 G。
    
- 一旦发现超时，会通过协作（如函数调用时的栈增长检查）或异步信号的方式强制将该 G 挂起，放回队列，让出执行权。
---

### 四、 为什么 GMP 比传统多线程高效？（面试结语）

在回答完细节后，用一段总结来体现你的技术视角：

1. **降低开销**：G 是用户态切换，不涉及内核态上下文切换，上下文仅需保存少量寄存器值，切换成本极低（约 200ns vs 线程的几微秒）。
    
2. **减少竞争**：引入 P 层的本地队列，使得大多数调度操作无需全局锁，大大降低了锁竞争。
    
3. **策略灵活**：通过 Work-Stealing 和 Hand Off 机制，在用户态实现了对任务的高度自适应和负载均衡。
## 五、Goroutine 的应用

考虑一个场景，需要从多个文件中读取数据，使用协程可以并发地进行文件读取，减少整体的 I/O 等待时间。这里通过模拟延迟来代表实际的文件读取操作。

```go
package main

import (
    "fmt"
    "time"
)

// readFile 现在将结果发送到通道中
func readFile(fileName string, ch chan<- string) {
    time.Sleep(time.Second)
    ch <- "模拟从 " + fileName + " 读取的数据"
}

func main() {
    fileNames := []string{"file1.txt", "file2.txt", "file3.txt"}
    // 创建一个带缓冲的 channel，容量等于文件数量
    ch := make(chan string, len(fileNames))

    for _, fileName := range fileNames {
        go readFile(fileName, ch)
    }

    var results []string
    // 从通道中收集结果
    for i := 0; i < len(fileNames); i++ {
        results = append(results, <-ch)
    }

    for _, result := range results {
        fmt.Println(result)
    }
}

// Go 的名言是：“不要通过共享内存来通信，而要通过通信来共享内存。”
```

## 六、总结

Goroutine 是 Go 语言并发编程的核心，掌握其正确使用方法对于编写高性能 Go 程序至关重要。

1. **简单易用** ：一个 `go` 关键字即可启动。
2. **轻量高效** ：内存占用小，切换成本低。
3. **需要同步** ：Goroutine 是异步执行的，主程序（ `main` ）需要等待它们完成，通常使用 `sync.WaitGroup` 或 Channel。
4. **注意安全** ：并发环境下要注意共享内存的安全问题（Data Race），并小心 Goroutine 泄漏。

本篇只介绍了协程的简单使用，真正的并发程序还是要结合 **通道 （channel）** 来实现。关于信道的内容，将在下一篇文章中介绍。
