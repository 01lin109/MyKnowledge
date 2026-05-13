---
title: "4.4 学习 Go 协程：Channel（通道） | Go 编程时光"
source: "https://go.sixue.work/golang/c04/c04_04.html"
author:
published: 2026-04-21
created: 2026-05-10
description: "基于 VitePress 的 Go 语言完整学习网站"
tags:
  - "clippings"
---
##  学习 Go 协程：Channel（通道）

`Channel` （通道）是 Go 语言并发编程中的核心概念。如果说 `goroutine` 是 Go 程序中的并发执行体，那么 `channel` 就是这些执行体之间安全通信的桥梁。

Go 语言遵循 **CSP (Communicating Sequential Processes)** 并发模型，其核心理念是： **通过通信来共享内存，而不是通过共享内存来通信** 。Channel 正是实现这一理念的关键机制。

## 一、Channel 简介与基本操作

### 1\. 为什么需要 Channel？
在并发场景中，多个 goroutine 可能需要访问同一块共享内存进行数据交换。Channel 提供了一种 **原生、安全、优雅** 的解决方案：
- 它是一种 **队列式** 数据结构，遵循 **FIFO（First In First Out，先进先出）** 原则。
- 自动保证了并发安全，无需手动加锁。
- 让数据在 goroutine 之间有序流动。
### 2\. 声明与创建
Channel 是 **引用类型** ，零值为 `nil` ；实际使用前要用 `make` 创建实例（与切片、映射类似）。
- **类型写法：** `chan 元素类型` （如 `chan int` 、 `chan []string` ）。
- **创建：** `make(chan 元素类型[, 缓冲容量])` 。省略第二个参数或传 `0` 为无缓冲；正整数为有缓冲。
```go
var ch1 chan int              // 仅声明，值为 nil，不能直接用来通信
ch1 = make(chan int)          // 先声明再初始化（较少用）

ch2 := make(chan int)         // 无缓冲（容量 0），最常见
ch3 := make(chan string, 5)   // 有缓冲，容量 5
```
### 3\. Channel 的核心操作
Channel 有三种基本操作： **发送** 、 **接收** 和 **关闭** 。发送和接收操作都使用 `<-` 箭头符号。

| 操作          | 语法            | 说明                       |
| ----------- | ------------- | ------------------------ |
| **发送**      | `ch <- value` | 把数据 `value` 发送到通道 `ch` 中 |
| **接收**      | `x := <-ch`   | 从通道 `ch` 接收数据并赋值给 `x`    |
| **接收（忽略值）** | `<-ch`        | 从通道接收数据但不使用该值            |
| **关闭**      | `close(ch)`   | 关闭通道（通常由发送方调用）           |

## 二、无缓冲 Channel（同步模式）

无缓冲 Channel 容量为 0，发送操作（ch <- value）和接收操作（value:= <-ch）会阻塞，直到对应的接收方或发送方准备好。

这使得发送和接收操作能够同步进行，常用于 Goroutine 之间的同步。 **常用于多个 goroutine 之间的同步** （例如互相等待就绪、传递「可以继续」的信号）。

![](https://go.sixue.work/images/channel1.png)

（无缓冲：发送与接收需配对，相当于「手递手」同步。）

**创建方式：**

```go
ch := make(chan int)        // 不指定容量，默认为 0
ch := make(chan int, 0)     // 显式指定容量为 0
```

**核心特性：**

- **容量为 0** ：不能存储任何元素，相当于一个"直通管道"。
- **必须配对** ：发送和接收操作必须同时就绪，否则会阻塞。
- **同步通信** ：发送方会等接收方，接收方会等发送方，实现了一种"握手"机制。

**形象比喻：** 就像两个人直接手递手传递东西，必须双方同时在场才能完成交接。

### 死锁示例

**❌ 错误示例：单 goroutine 中操作无缓冲 Channel**

```go
func main() {
    ch := make(chan int)
    ch <- 10       // ⚠️ 永远阻塞在这里，没有接收方
    fmt.Println("发送成功")
}
```

**运行结果：**

```
fatal error: all goroutines are asleep - deadlock!
```

**原因分析：** 无缓冲 Channel 要求发送和接收同时就绪。这里只有发送操作，没有接收操作，主 goroutine 永远等待，形成死锁。

### 发送操作

当向无缓冲通道发送数据时，如果没有其他 Goroutine 在接收数据，发送操作会一直阻塞，直到有接收方准备好。

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan int) // 无缓冲通道

    // 启动一个Goroutine，2秒后才开始接收数据
    go func() {
        time.Sleep(2 * time.Second) // 模拟延迟，让发送方先执行
        fmt.Println("接收方准备接收数据")
        val := <-ch // 接收数据
        fmt.Println("接收方收到数据:", val)
    }()

    fmt.Println("发送方准备发送数据...")
    ch <- 100 // 发送操作：此时接收方未准备好，会阻塞2秒
    fmt.Println("发送方完成数据发送") // 2秒后才会执行
}
```

### 接收操作

当从无缓冲通道接收数据时，如果没有其他 Goroutine 在发送数据，接收操作会一直阻塞，直到有发送方准备好。这使得发送和接收操作能够同步进行。

```
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan int) // 无缓冲通道

    // 启动一个Goroutine，2秒后才开始发送数据
    go func() {
        time.Sleep(2 * time.Second) // 模拟延迟，让接收方先执行
        fmt.Println("发送方准备发送数据")
        ch <- 200 // 发送数据
        fmt.Println("发送方完成数据发送")
    }()

    fmt.Println("接收方准备接收数据...")
    val := <-ch // 接收操作：此时发送方未准备好，会阻塞2秒
    fmt.Println("接收方收到数据:", val) // 2秒后才会执行
}
```

## 三、有缓冲 Channel（异步模式）

有缓冲 Channel 在创建时指定容量（大于 0），缓冲区未满时发送、非空时接收可以 **不必** 配对完成，阻塞规则与无缓冲不同。

![](https://go.sixue.work/images/channel2.png)

（有缓冲：未满可先发送、非空可先接收，缓冲区起到暂存作用。）

**创建方式：**

```go
ch := make(chan int, 5)  // 创建容量为 5 的缓冲通道
```

**核心特性：**

- **有存储空间** ：可以容纳指定数量的元素（容量 > 0）。
- **异步通信** ：发送和接收不需要同时进行，缓冲区起到"缓冲池"的作用。
- **阻塞规则** ：
	- 发送：缓冲区未满时立即返回，满了才阻塞。
		- 接收：缓冲区非空时立即返回，空了才阻塞。

**常用函数：**
- `cap(ch)` - 获取 Channel 的总容量
- `len(ch)` - 获取 Channel 当前存储的元素数量
**形象比喻：** 就像一个邮箱，可以先把信放进去（发送），收件人稍后再取（接收），不需要双方同时在场。
### 死锁解决示例

同样是上面死锁的例子，修改如下：

```go
package main

import "fmt"

func main() {
    ch := make(chan int, 1) // 创建一个容量为1的有缓冲区通道
    ch <- 10
    fmt.Println("发送成功")
}
```

运行后不会报错，因为发送操作会立即返回，不会阻塞。

### 基本用法

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan int, 3) // 创建容量为3的缓冲 channel

    fmt.Printf("[初始] 长度: %d，容量: %d\n", len(ch), cap(ch))

    // 连续发送3个数据，不会阻塞
    ch <- 1
    fmt.Printf("[发送1] 长度: %d\n", len(ch))
    ch <- 2
    ch <- 3
    fmt.Printf("[发送2,3] 长度: %d\n", len(ch))

    // 再发送将阻塞，直到缓冲区有空间
    go func() {
        fmt.Println("尝试发送4（会阻塞，等待缓冲区有空间）...")
        ch <- 4
        fmt.Println("成功发送4")
    }()

    time.Sleep(1 * time.Second) // 演示阻塞，等待 goroutine 启动

    val := <-ch // 接收一个，释放空间
    fmt.Printf("[接收] 拿到: %d，长度: %d\n", val, len(ch))

    time.Sleep(1 * time.Second) // 再等一下，保证 goroutine 完成
    fmt.Printf("[最后] 缓冲区长度: %d\n", len(ch))
}
```
**运行重点：**
- 前 3 次发送不阻塞（缓冲区未满）。
- 第 4 次发送时缓冲区已满，必须等待接收方取出数据后才能继续。
- 通过 `len(ch)` 和 `cap(ch)` 可以实时监控缓冲区状态。

## 四、关闭 Channel

当确定 Channel 中不会再有新数据发送时，发送方应该调用 `close(ch)` 来关闭 Channel，通知接收方"数据发送完毕"。

**示例：**

```
package main

import (
    "fmt"
)

func main() {
    ch := make(chan int)
    go func() {
        for i := 0; i < 5; i++ {
            ch <- i
        }
        close(ch)
    }()
    for value := range ch {
        fmt.Println("Received:", value)
    }
    fmt.Println("所有数据接收完毕")
}
```

**工作原理：** `close` 表示「不会再发送」； **已发送尚未被取走的数据仍可读完** 。

`close` 要 **谨慎** ：核心原则是 **谁负责发送数据，谁负责关闭 channel** 。接收方一般不要关「还在被写入」的数据 channel，否则既不符合语义，也容易和别的 goroutine **重复 `close`** 。

### 需要手动关闭的情况

当发送方 **已经确定不会再发送** ，并且需要 **明确通知接收方「没有更多数据 / 任务结束」 **时，应由发送方在发完后 `close(ch)` 。否则接收方若用 `for range ch` 或一直读，可能** 永远阻塞** 。

- **生产者-消费者** ：生产者发完所有数据后 `close` ；消费者用 `for range` 读完自动退出，或用 `value, ok := <-ch` ，在 **`ok == false`** 时结束。
```
package main

import "fmt"

func producer(ch chan<- int) {
    for i := 0; i < 3; i++ {
        ch <- i
    }
    close(ch) // 发送方发完再关
}

func consumer(ch <-chan int) {
    for num := range ch {
        fmt.Println("Received:", num)
    }
}

func main() {
    ch := make(chan int)
    go producer(ch)
    consumer(ch)
}
```
- **「任务完成」类信号** ：channel 只用来传「结束」语义时，同样由 **结束工作的一方** 关闭。常用 `chan struct{}` ，做完事后 `close(done)` ，等待方 `<-done` 即可继续。
```
package main

import (
    "fmt"
    "time"
)

func main() {
    done := make(chan struct{})
    go func() {
        time.Sleep(1 * time.Second)
        close(done)
    }()
    <-done
    fmt.Println("任务已完成")
}
```

### 不必手动关闭的情况

- **临时 channel、用完即无引用** ：例如只发一次、收一次，之后没有任何 goroutine 再持有该 channel，由 **GC 回收** 即可，不必形式化 `close` 。
```
package main

import "fmt"

func main() {
    ch := make(chan int)
    go func() { ch <- 10 }()
    fmt.Println(<-ch) // 可不 close
}
```
- **多个发送方共用一个 channel** ：无法约定「谁最后一个发送、谁负责 `close` 」时，若多人 `close` 会 **panic（重复关闭）** 。通常 **不要 close 数据 channel** ，改用 **单独的控制 channel** （如 `quit` ），只由 **一处** `close(quit)` ，各发送方在 `select` 里 `case <-quit: return` （见第六节）。
- **不需要「流结束」语义** ：单次传值、握手即结束，没有长期 `for range` ，关不关对逻辑帮助不大， **可以不关** 。

### 必须遵守的规则

- **接收方通常不要 `close` 数据 channel** （关闭是发送方的收尾）。
- **同一 channel 只能 `close` 一次** ； **`close` 后禁止再发送** ； **不要 `close(nil)`** ——具体后果见下表。
- channel 未关闭 **不等于** 内存泄漏；但若消费者用 `for range` **等结束** ，生产者 **既不再发也不 `close`** ，可能造成 **goroutine 长期阻塞** ，要和整体生命周期一起设计。

### 关闭后的收发行为

关闭只表示 **不会再有新发送** ， **缓冲里或已在途的数据仍可被接收完** 。之后继续接收时：若还有未读数据则读到 **真实值** ；若没有未读数据，则 `value, ok := <-ch` 得到 **零值** 且 **`ok == false`** ， **不阻塞** ； `for range` 也会在读完已有数据后结束。

记住： **关闭 ≠ 立刻读不到** ； **零值 + `ok == false`** 才表示「已关闭且无数据」。

| 操作 | 结果 |
| --- | --- |
| 向已关闭的 Channel **发送** | **panic** |
| 从已关闭的 Channel **接收** （仍有未读数据） | 正常读到数据 |
| 从已关闭的 Channel **接收** （无未读数据） | **零值** ， `ok == false` ， **不阻塞** |
| 对已关闭的 Channel **再次 close** | **panic** |
| `close(nil)` | **panic** |

## 五、单向 Channel：权限控制

在实际开发中，我们常常希望限制 Channel 的操作权限，防止误用。Go 提供了 **单向 Channel** 来实现这一目的。

### 1\. 类型定义

| 类型 | 语法 | 说明 | 用途 |
| --- | --- | --- | --- |
| **只发送通道** | `chan<- int` | 只能向通道发送数据 | 作为生产者的参数 |
| **只接收通道** | `<-chan int` | 只能从通道接收数据 | 作为消费者的参数 |
| **双向通道** | `chan int` | 既可发送也可接收（默认） | 创建时的类型 |

### 2\. 典型用法

单向 Channel 通常用于 **函数参数** ，明确该函数对 Channel 的操作意图，提高代码安全性和可读性。

**示例：**

```go
package main

import "fmt"

// 生产者函数：只能发送
func producer(out chan<- int) {
    for i := 0; i < 3; i++ {
        out <- i
    }
    close(out)
}

// 消费者函数：只能接收
func consumer(in <-chan int) {
    for num := range in {
        fmt.Println("消费:", num)
    }
}

func main() {
    ch := make(chan int) // 创建双向通道
    go producer(ch)      // 传入时自动转为只发送通道
    consumer(ch)         // 传入时自动转为只接收通道
}
```

### 3\. 转换规则

✅ **允许：** 双向通道 → 单向通道（自动转换）  
❌ **禁止：** 单向通道 → 双向通道（编译错误）

**好处：** 编译期就能检测出错误的操作，比如在消费者函数中误写了发送操作。

## 六、select：多路复用

在实际应用中，我们常常需要同时监听多个 Channel。如果用普通的方式逐个读取，只要某个 Channel 阻塞，整个程序就卡住了。

为了解决这个问题，Go 提供了 `select` 语句，它是 Channel 的"多路复用器"。

### 1\. 什么是 select？

`select` 可以看作是专门为 Channel 设计的 `switch` 语句：

- 可以同时监听多个 Channel 的读写操作。
- 哪个 Channel 先准备好，就执行哪个 case。
- 如果多个同时准备好，随机选择一个（保证公平）。

**语法格式：**

```go
select {
case <-ch1:
    // 如果 ch1 成功读到数据，则执行该分支
case data := <-ch2:
    // 如果 ch2 成功读到数据，则执行该分支
case ch3 <- data:
    // 如果成功向 ch3 写入数据，则执行该分支
default:
    // 如果上面都没有成功，则进入 default 分支（可选）
}
```

### 2\. select 的执行规则

| 情况 | 行为 |
| --- | --- |
| **没有 case 准备好，且无 default** | 阻塞等待，直到某个 case 可以执行 |
| **只有一个 case 准备好** | 执行该 case |
| **多个 case 同时准备好** | **随机** 选择一个执行（公平调度） |
| **没有 case 准备好，但有 default** | 立即执行 default，不阻塞 |

### 3\. 实战场景

下面两个用法在工程里最常见，也最容易对照「多路等待」来理解。

#### 场景一：竞争模式（谁先到用谁）

两个数据源（例如两次请求、两个缓存回填）只需 **先就绪** 的那一路结果： `select` 会阻塞到 **至少一个** `case` 可执行时再进入对应分支；若多个 `case` 在同一时刻都可执行，则 **随机** 选一个（公平，并非固定顺序）。

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)

    go func() {
        time.Sleep(1 * time.Second)
        ch1 <- "数据源 1"
    }()
    go func() {
        time.Sleep(2 * time.Second)
        ch2 <- "数据源 2"
    }()

    fmt.Println("开始监听...")
    select {
    case msg1 := <-ch1:
        fmt.Println("接收到:", msg1)
    case msg2 := <-ch2:
        fmt.Println("接收到:", msg2)
    }
    fmt.Println("main 结束")
}
```

#### 场景二：超时控制

用 `time.After` 返回的通道当作「定时信号」，与业务通道一起放进 `select` ，避免一直卡在接收上。

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan int)

    go func() {
        time.Sleep(3 * time.Second)
        ch <- 100
    }()

    fmt.Println("开始等待数据...")
    select {
    case val := <-ch:
        fmt.Println("收到数据:", val)
    case <-time.After(2 * time.Second):
        fmt.Println("错误: 操作超时！")
    }
}
```

生产环境里超时、取消常与 **`context`** （如 `context.WithTimeout` ）配合，比单独拼 `time.After` 更易在整条调用链上传递截止时间。

> **延伸：** 试探性读通道可加 `default` 避免阻塞； `for` + `select` 同时监听业务与退出通道，可控制 goroutine 生命周期。跳出外层循环见下文「注意事项」中的标签写法。

### 4\. 注意事项

**⚠️ 空 select 会永久阻塞：**

```go
select {} // 当前 goroutine 永远阻塞，相当于死锁
```

**⚠️ break 只跳出 select，不跳出外层循环：**

```go
for {
    select {
    case <-ch:
        break // 只跳出 select，循环继续
    }
}
```

如果需要跳出外层循环，使用 **标签** ：

```go
Loop:
    for {
        select {
        case <-ch:
            break Loop // 跳出外层循环
        }
    }
```

## 七、循环接收：优雅地读取 Channel

### 1\. 使用 for range

这是最优雅、最常用的方式。 `for range` 会自动处理 Channel 的状态：

- **有数据** → 正常读取
- **无数据且未关闭** → 阻塞等待
- **已关闭且数据读完** → 自动退出循环

**示例：**

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan int)

    // 生产者
    go func() {
        for i := 0; i < 3; i++ {
            ch <- i
            time.Sleep(100 * time.Millisecond)
        }
        close(ch) // 任务完成后，必须关闭 Channel
        fmt.Println("发送方：数据发送完毕，关闭 Channel")
    }()

    // 消费者
    fmt.Println("开始接收...")
    for data := range ch {
        fmt.Println("接收到:", data)
    }
    fmt.Println("接收方：Channel 已关闭，循环退出")
}
```

⚠️ **重要提示：** 如果发送方忘记 `close(ch)` ， `for range` 会永久阻塞，最终导致死锁错误。

### 2\. 使用 for + ok 手动判断

这种方式更灵活，需要手动检查 Channel 状态。

**语法：** `value, ok := <-ch`

- `ok == true` → 成功读到数据
- `ok == false` → Channel 已关闭且无数据

**示例：**

```go
package main

import "fmt"

func main() {
    ch := make(chan int, 2)

    ch <- 10
    ch <- 20
    close(ch)

    for {
        data, ok := <-ch
        if !ok {
            // Channel 关闭且无数据，退出循环
            fmt.Println("通道已关闭")
            break
        }
        fmt.Println("接收到:", data)
    }
}
```

**适用场景：** 当需要在循环中结合 `select` 或其他复杂逻辑时，手动判断比 `for range` 更灵活。

## 八、最佳实践与常见陷阱

### 1\. 最佳实践

✅ **谁创建谁关闭**  
Channel 的所有者（通常是发送方）负责关闭它。接收方不应该关闭 Channel。

✅ **使用单向 Channel**  
在函数参数中使用 `<-chan` 或 `chan<-` 明确职责，防止误操作，提高代码可读性。

✅ **优先使用 `for range`**  
遍历 Channel 时首选 `for range` ，它简洁且能自动处理关闭状态。

**组合示范：** 单向通道 + 生产/消费分工见 **第五节** 示例；此处只强调一种常见写法——生产者里用 **`defer close(out)`** ，即使中途 `return` 或 `panic` 走 `recover` 路径，也尽量保证通道被关闭，避免消费者 `for range` 永远挂起：

```go
func producer(out chan<- int) {
    defer close(out)
    for i := 0; i < 5; i++ {
        out <- i
    }
}
```

（完整 `main` 与消费者循环与第五节相同，此处不重复贴全文件。）

### 2\. 常见陷阱

#### ❌ 陷阱 1：忘记关闭 Channel

**现象：** 消费者使用 `for range` 持续等待，但生产者忘记关闭 Channel。  
**后果：** Goroutine 永久阻塞，导致泄漏或死锁。

**解决：** 生产者任务完成后务必调用 `close(ch)` 。

#### ❌ 陷阱 2：向已关闭的 Channel 发送

**现象：** Channel 已经关闭，但仍有代码试图发送数据。  
**后果：** 立即触发 `panic: send on closed channel` 。

**解决：** 确保 Channel 关闭后不再有发送操作（通常通过代码逻辑保证）。

#### ❌ 陷阱 3：nil Channel 的行为

未初始化的 Channel 值为 `nil` ，操作 `nil` Channel 会有特殊行为：

| 操作 | 结果 |
| --- | --- |
| 向 `nil` Channel 发送 | 永久阻塞 |
| 从 `nil` Channel 接收 | 永久阻塞 |
| 关闭 `nil` Channel | panic |

```go
var ch chan int  // ch 是 nil
// ch <- 1       // ❌ 永久阻塞
// <-ch          // ❌ 永久阻塞
// close(ch)     // ❌ panic
```

## 九、进阶技巧：用 Channel 实现信号量（选读）

> 本节帮助理解「通道即并发原语」的一种惯用法；业务代码里限制并发更常见的是 **`golang.org/x/sync/semaphore`** 或 `errgroup` 等组合，不必手写本节模式。

虽然 Go 提供了 `sync.Mutex` 互斥锁，但利用 Channel 的阻塞特性，可以实现限流式的并发控制。

### 原理：用缓冲通道计数

与「空缓冲 + 先接收再发送」的另一类信号量写法不同，这里用 **带缓冲的 channel** ：容量为 N 时，通道里 **最多能暂存 N 次发送** ；多出来的发送会阻塞，从而把同时在跑的任务数限制在 N。

- `make(chan struct{}, N)` → 最多 N 个 `send` 在未配套 `receive` 前都能成功（表示 N 个并发名额被占满）。
- **获取许可** = `sem <- struct{}{}` （缓冲已满则阻塞，即已达并发上限）。
- **释放许可** = `<-sem` （取走一个占位，腾出名额）。

### 典型应用：控制最大并发数（信号量）

这是 Channel 相比 Mutex 更擅长的场景：限制同时运行的 Goroutine 数量。

```
package main

import (
    "fmt"
    "time"
)

func worker(id int, sem chan struct{}) {
    // 1. 获取许可（缓冲已满则阻塞等待）
    sem <- struct{}{}
    fmt.Printf("Worker %d: 正在运行...\n", id)
    
    time.Sleep(1 * time.Second) // 模拟耗时任务
    
    fmt.Printf("Worker %d: 完成\n", id)
    // 2. 释放许可
    <-sem
}

func main() {
    // 容量为 3 → 最多 3 个 worker 同时运行
    sem := make(chan struct{}, 3)

    for i := 1; i <= 10; i++ {
        go worker(i, sem)
    }

    time.Sleep(5 * time.Second) // 等待所有任务完成
}
```

**运行效果：** 虽然启动了 10 个 goroutine，但最多只有 3 个同时在运行，其他的会排队等待。

## 十、总结

Channel 是 Go 并发编程的核心，也是区别于其他语言的关键特性。掌握 Channel 对于编写高质量的 Go 程序至关重要。

### 核心知识点回顾

| 概念 | 要点 |
| --- | --- |
| **CSP 模型** | 通过通信来共享内存，而非通过共享内存来通信 |
| **无缓冲 Channel** | 容量为 0，强制发送和接收同步，适合"握手"场景 |
| **有缓冲 Channel** | 有存储空间，异步通信，适合解耦和缓冲 |
| **单向 Channel** | 限制权限（只发送/只接收），提高安全性 |
| **循环接收** | 使用 `for range` 或 `value, ok := <-ch` 优雅地读取 Channel |
| **select 语句** | 同时监听多个 Channel；常用场景含超时、多路等待（竞争）、配合 `default` 非阻塞等 |
| **关闭规则** | 发送方关闭，关闭后不能再发送，可继续接收 |

掌握了 Channel，你就掌握了 Go 并发编程的精髓。接下来的章节我们将学习更多的并发工具和模式。