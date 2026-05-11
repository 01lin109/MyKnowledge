---
title: "4.5 学习 Go 协程：WaitGroup | Go 编程时光"
source: "https://go.sixue.work/golang/c04/c04_05.html"
author:
published: 2026-04-21
created: 2026-05-11
description: "基于 VitePress 的 Go 语言完整学习网站"
tags:
  - "clippings"
---
## WaitGroup

在前两篇文章里，我们学习了 [[goroutine（协程）]] 和 [[Channel（通道）]] 的内容，里面有很多例子，当时为了保证 main goroutine 在所有的 goroutine 都执行完毕后再退出，使用了 time.Sleep 这种简单的方式。

但在实际开发中，开发人员是无法预知，所有的 goroutine 需要多长的时间才能执行完毕，sleep 多了主程序就阻塞了， sleep 少了有的子协程的任务就没法完成。因此，使用time.Sleep 是一种极不推荐的方式，仅为演示，今天介绍如何优雅的处理这种情况。

## 一、WaitGroup 简介

`sync.WaitGroup` 是一个用于等待一组 goroutine 完成执行的同步工具，它能确保主 goroutine（或其他协调者）阻塞等待，直到所有注册的子 goroutine 都执行完毕后再继续运行。

简单来说，它的作用是： **等待一组 Goroutine 执行完成。**

**核心作用：**

想象一个场景：主程序启动了多个子任务（goroutine），需要等所有子任务都做完后，再进行汇总或打印结果。如果没有 WaitGroup，主程序可能会在子任务完成前就退出，导致子任务的结果丢失或未执行完。

WaitGroup 就是为解决这个问题而生的，它通过简单的 API 实现 “等待所有子任务结束” 的功能。

**核心思想：**

1. **启动 goroutine 前：** 设置或增加计数器的值。
2. **子 goroutine 中：** 任务完成后，减少计数器的值。
3. **主 goroutine 中：** 阻塞等待，直到计数器归零。

## 二、三个核心方法

`sync.WaitGroup` 内部维护了一个计数器，你只需要通过以下三个方法来操作它：

| 方法 | 作用 | 描述 |
| --- | --- | --- |
| **`Add(delta int)`** | 计数器 += `delta` | 注册需要等待的 goroutine 数量（n 为子任务总数）。通常在启动子 goroutine 前调用。 |
| **`Done()`** | 减少计数 | 每个子 goroutine 执行完毕后调用，告知 WaitGroup“我完成了”（相当于 Add(-1)）。通常用 defer 语句确保一定会被调用。 |
| **`Wait()`** | 阻塞等待 | 主 goroutine 调用，会阻塞等待，直到所有注册的子 goroutine 都调用了 Done()（即等待的数量减为 0），才会继续执行后续代码。 |

## 三、基本使用示例

以下是一个完整的示例，展示了如何使用 `WaitGroup` 来确保主程序等待所有子 goroutine 打印完数字后再退出。

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func task(id int, wg *sync.WaitGroup) {
    defer wg.Done() // 子任务结束时调用 Done()
    fmt.Printf("子任务 %d 开始\n", id)
    time.Sleep(time.Second) // 模拟任务执行
    fmt.Printf("子任务 %d 结束\n", id)
}

func main() {
    var wg sync.WaitGroup

    // 注册 3 个需要等待的子任务
    wg.Add(3)

    // 启动 3 个子 goroutine
    for i := 1; i <= 3; i++ {
        go task(i, &wg)
    }

    // 主 goroutine 等待所有子任务完成
    wg.Wait()
    fmt.Println("所有子任务已完成，主程序退出")
}
```

**输出结果：**

```
子任务 1 开始
子任务 2 开始
子任务 3 开始
（等待 1 秒后）
子任务 1 结束
子任务 2 结束
子任务 3 结束
所有子任务已完成，主程序退出
```

## 四、使用注意事项

### 1\. Add() 必须在 Wait() 之前调用

你必须在启动 goroutine **之前** 或在确定 goroutine 已启动但尚未调用 `Done()` **之前** 调用 `Add()` 。

### 2\. Done() 必须与 Add() 匹配

`Done()` 调用必须严格匹配 `Add()` 设置的计数。

如果 `Done()` 或 `Add(-1)` 使得计数器小于零，程序可能触发 `panic` ，或者导致 `Wait()` 提前返回。

### 3\. 推荐使用 defer 调用 Done()

为了保证 goroutine 无论成功或失败都能正确通知 `WaitGroup` ，强烈推荐将 `wg.Done()` 放在 goroutine 函数的开头，使用 `defer` 关键字修饰。

```go
func worker(id int, wg *sync.WaitGroup) {
    defer wg.Done() // 确保无论发生什么，计数器都会减一
    // ... 任务代码
}
```

### 4\. 传参时使用指针 \*sync.WaitGroup

虽然 `WaitGroup` 是一个结构体，但在函数传参时，应传入其指针 `*sync.WaitGroup` 。

**原因：** 传入值拷贝会导致每个 goroutine 接收到的是 `WaitGroup` 的副本，对副本调用的 `Done()` 无法影响到主 goroutine 中 `Wait()` 所依赖的原始 `WaitGroup` 计数器，从而造成程序提前退出或永久阻塞（死锁）。

**正确写法：**

```go
func main() {
    var wg sync.WaitGroup
    wg.Add(1)
    go worker(&wg) // 传递指针
    wg.Wait()
}

func worker(wg *sync.WaitGroup) { // 接收指针
    defer wg.Done()
}
```

### 5\. 不可重复使用

一个 `WaitGroup` 完成一次等待后，不建议重复使用（可能因内部状态未重置导致异常），如需再次等待，建议创建新的 `WaitGroup。`

## 五、WaitGroup 与 Channel 的区别

`WaitGroup` 和 `Channel` 都可以实现并发同步，但它们的作用和关注点不同：

| 特性 | `sync.WaitGroup` | `channel` |
| --- | --- | --- |
| **主要作用** | **同步等待** 一组任务完成。 | **数据通信** ，在 goroutine 间安全地传递数据。 |
| **功能** | 计数器模型，仅用于 **通知** 任务完成状态。 | 队列模型，用于 **传输** 数据，也具备同步能力（阻塞）。 |
| **通信方向** | 单向：从子 goroutine 通知主 goroutine。 | 双向：可在任意两个 goroutine 间发送和接收数据。 |
| **关注点** | **任务数量** ，确保所有任务都执行完成。 | **数据流转** ，确保数据的安全和有序传输。 |

**总结：**

- 如果你的需求是 **等待所有并发任务完成** ，而不涉及任务之间的数据传递，请使用 **`sync.WaitGroup`** 。
- 如果你的需求是 **在并发任务之间安全地交换数据** ，请使用 **`channel`** 。