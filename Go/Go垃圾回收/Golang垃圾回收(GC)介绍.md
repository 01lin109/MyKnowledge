---
title: "Golang垃圾回收(GC)介绍"
source: "https://liangyaopei.github.io/2021/01/02/golang-gc-intro/"
author:
  - "[[LiangYaopei]]"
published: 2021-01-02
created: 2026-05-14
description: "常见的GC算法引用计数法根据对象自身的引用计数来回收，当引用计数归零时进行回收，但是计数频繁更新会带来更多开销，且无法解决循环引用的问题。  优点：简单直接，回收速度快 缺点：需要额外的空间存放计数，无法处理循环引用的情况；"
tags:
  - "clippings"
---
## 常见的GC算法

## 引用计数法（不用了）

根据对象自身的引用计数来回收，当引用计数归零时进行回收，但是计数频繁更新会带来更多开销，且无法解决循环引用的问题。

- 优点：简单直接，回收速度快
- 缺点：需要额外的空间存放计数，无法处理循环引用的情况；

## 标记清除法
[[JVM垃圾回收#垃圾收集算法]] 逻辑基本一致，实现机制不同
标记出所有不需要回收的对象，在标记完成后统一回收掉所有未被标记的对象。  

- 优点：简单直接，速度快，适合可回收对象不多的场景
- 缺点：会造成不连续的内存空间（内存碎片），导致有大的对象创建的时候，明明内存中总内存是够的，但是空间不是连续的造成对象无法分配；

## 复制法

复制法将内存分为大小相同的两块，每次使用其中的一块，当这一块的内存使用完后，将还存活的对象复制到另一块去，然后再把使用的空间一次清理掉  
![copy_method](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/copy_method.png)

- 优点：解决了内存碎片的问题，每次清除针对的都是整块内存，但是因为移动对象需要耗费时间，效率低于标记清除法；
- 缺点：有部分内存总是利用不到，资源浪费，移动存活对象比较耗时，并且如果存活对象较多的时候，需要担保机制确保复制区有足够的空间可完成复制；

## 标记整理

标记过程同标记清除法，结束后将存活对象压缩至一端，然后清除边界外的内容  
![mark_tidy](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/mark_tidy.png)

- 优点：解决了内存碎片的问题，也不像标记复制法那样需要担保机制，存活对象较多的场景也使适用；
- 缺点：性能低，因为在移动对象的时候不仅需要移动对象还要维护对象的引用地址，可能需要对内存经过几次扫描才能完成；

## 分代式

将对象根据存活时间的长短进行分类，存活时间小于某个值的为年轻代，存活时间大于某个值的为老年代，永远不会参与回收的对象为永久代。并根据分代假设（如果一个对象存活时间不长则倾向于被回收，如果一个对象已经存活很长时间则倾向于存活更长时间）对对象进行回收。

## Golang的垃圾回收（GC）算法

Golang的垃圾回收（GC）算法使用的是无无分代（对象没有代际之分）、不整理（回收过程中不对对象进行移动与整理）、并发（与用户代码并发执行）的三色标记清扫算法。原因在于：

- 对象整理的优势是解决内存碎片问题以及“允许”使用顺序内存分配器。但 Go 运行时的分配算法基于 `tcmalloc` ，基本上没有碎片问题。 并且顺序内存分配器在多线程的场景下并不适用。Go 使用的是基于 `tcmalloc` 的现代内存分配算法，对对象进行整理不会带来实质性的性能提升。
- 分代 `GC` 依赖分代假设，即 `GC` 将主要的回收目标放在新创建的对象上（存活时间短，更倾向于被回收），而非频繁检查所有对象。
- Go 的编译器会通过逃逸分析将大部分新生对象存储在栈上（栈直接被回收），只有那些需要长期存在的对象才会被分配到需要进行垃圾回收的堆中。也就是说，分代 `GC` 回收的那些存活时间短的对象在 Go 中是直接被分配到栈上，当 `goroutine` 死亡后栈也会被直接回收，不需要 `GC` 的参与，进而分代假设并没有带来直接优势。
- Go 的垃圾回收器与用户代码并发执行，使得 STW 的时间与对象的代际、对象的 size 没有关系。Go 团队更关注于如何更好地让 GC 与用户代码并发执行（使用适当的 CPU 来执行垃圾回收），而非减少停顿时间这一单一目标上。

## 三色标记法原理

三色标记法将对象分为三类，并用不同的颜色相称：

- 白色对象（可能死亡）：未被回收器访问到的对象。在回收开始阶段，所有对象均为白色，当回收结束后，白色对象均不可达。
- 灰色对象（波面）：已被回收器访问到的对象，但回收器需要对其中的一个或多个指针进行扫描，因为他们可能还指向白色对象。
- 黑色对象（确定存活）：已被回收器访问到的对象，其中所有字段都已被扫描，黑色对象中任何一个指针都不可能直接指向白色对象。

标记过程如下：

（1）起初所有的对象都是白色的；

（2）从根对象出发扫描所有可达对象，标记为灰色，放入待处理队列；

（3）从待处理队列中取出灰色对象，将其引用的对象标记为灰色并放入待处理队列中，自身标记为黑色；

（4）重复步骤（3），直到待处理队列为空，此时白色对象即为不可达的“垃圾”，回收白色对象；

![three_color](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/three_color.gif)

> 根对象在垃圾回收的术语中又叫做根集合，它是垃圾回收器在标记过程时最先检查的对象，包括：

1. 全局变量：程序在编译期就能确定的那些存在于程序整个生命周期的变量。
2. 执行栈：每个 goroutine 都包含自己的执行栈，这些执行栈上包含栈上的变量及指向分配的堆内存区块的指针。
3. 寄存器：寄存器的值可能表示一个指针，参与计算的这些指针可能指向某些赋值器分配的堆内存区块。

## 屏障机制

### STW

STW 可以是Stop The World的缩写，也可以是Start The World的缩写。通常意义上指的是从Stop The World到Start The World这一段时间间隔。垃圾回收过程中为了保证准确性、防止无止境的内存增长等问题而不可避免的需要停止赋值器进一步操作对象图以完成垃圾回收。STW时间越长，对用户代码造成的影响越大。

### No STW 存在的问题

假设下面的场景，已经被标记为灰色的对象2，未被标记的对象3被对象2用指针p引用；此时已经被标记为黑色的对象4创建指针q 指向未被标记的对象3，同时对象2将指针p移除；对象4已经被标记为黑色，对象3未被引用，对象2删除与对象3的引用，导致最后对象3被误清除；

![no_STW_1](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/no_STW_1.png)

![no_STW_2](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/no_STW_2.png)

![no_STW_3](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/no_STW_3.png)

![no_STW_4](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/no_STW_4.png)

![no_STW_5](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/no_STW_5.png)

- 垃圾回收的原则是不应出现对象的丢失，也不应错误的回收还不需要回收的对象。如果同时满足下面两个条件会破坏回收器的正确性：
	- 条件 1: 赋值器修改对象图，导致某一黑色对象引用白色对象；（通俗的说就是A突然持有了B的指针，而B在并发标记的过程中已经被判定为白色对象要被清理掉的）
		- 条件 2: 从灰色对象出发，到达白色对象且未经访问过的路径被赋值器破坏；（通俗的说就是A持有B的指针，这个持有关系被释放）

只要能够避免其中任何一个条件，则不会出现对象丢失的情况，因为：

- 如果条件 1被避免，则所有白色对象均被灰色对象引用，没有白色对象会被遗漏；
- 如果条件 2 被避免，即便白色对象的指针被写入到黑色对象中，但从灰色对象出发，总存在一条没有访问过的路径，从而找到到达白色对象的路径，白色对象最终不会被遗漏。

**可能的解决方法：** 整个过程STW，浪费资源，且对用户程序影响较大，由此引入了 **屏障机制** ；

### 屏障机制

把回收器视为对象，把赋值器视为影响回收器这一对象的实际行为（即影响 GC 周期的长短），从而引入赋值器的颜色：

- 黑色赋值器：已经由回收器扫描过，不会再次对其进行扫描。
- 灰色赋值器：尚未被回收器扫描过或尽管已经扫描过，但仍需要重新扫描。

#### 插入屏障（Dijkstra）- 灰色赋值器

写入前，对指针所要指向的对象进行着色

```
// 灰色赋值器 Dijkstra 插入屏障
func DijkstraWritePointer(slot *unsafe.Pointer, ptr unsafe.Pointer) {
    shade(ptr) //先将新下游对象 ptr 标记为灰色
    *slot = ptr
}

//说明：
添加下游对象(当前下游对象slot, 新下游对象ptr) {   
  //step 1
  标记灰色(新下游对象ptr)   
  
  //step 2
  当前下游对象slot = 新下游对象ptr                    
}

//场景：
A.添加下游对象(nil, B)   //A 之前没有下游， 新添加一个下游对象B， B被标记为灰色
A.添加下游对象(C, B)     //A 将下游对象C 更换为B，  B被标记为灰色
```

> 避免条件1（ 赋值器修改对象图，导致某一黑色对象引用白色对象；）因为在对象A 引用对象B 的时候，B 对象被标记为灰色

Dijkstra 插入屏障的好处在于可以立刻开始并发标记。但存在两个缺点：

- 由于 Dijkstra 插入屏障的“保守”，在一次回收过程中可能会残留一部分对象没有回收成功，只有在下一个回收过程中才会被回收；
- 在标记阶段中，每次进行 **指针赋值操作** 时，都需要 **引入写屏障** ，这无疑会增加大量性能开销；为了避免造成性能问题， `Go` 团队在最终实现时， **没有为所有栈上的指针写操作，启用写屏障** ，而是当发生栈上的写操作时，将栈标记为灰色，但此举产生了 **灰色赋值器** ，将会需要标记终止阶段 STW 时对这些栈进行 **重新扫描** 。

![insert_barrier_1](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/insert_barrier_1.png)

![insert_barrier_2](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/insert_barrier_2.png)

![insert_barrier_3](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/insert_barrier_3.png)

![insert_barrier_4](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/insert_barrier_4.png)

![insert_barrier_5](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/insert_barrier_5.png)

![insert_barrier_6](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/insert_barrier_6.png)

![insert_barrier_7](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/insert_barrier_7.png)

![insert_barrier_8](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/insert_barrier_8.png)

![insert_barrier_9](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/insert_barrier_9.png)

![insert_barrier_10](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/insert_barrier_10.png)

#### 删除屏障 （Yuasa）- 黑色赋值器

写入前，对指针所在对象进行着色

```
// 黑色赋值器 Yuasa 屏障
func YuasaWritePointer(slot *unsafe.Pointer, ptr unsafe.Pointer) {
    shade(*slot) 先将*slot标记为灰色
    *slot = ptr
}

//说明：
添加下游对象(当前下游对象slot， 新下游对象ptr) {
  //step 1
  if (当前下游对象slot是灰色 || 当前下游对象slot是白色) {
          标记灰色(当前下游对象slot)     //slot为被删除对象， 标记为灰色
  }  
  //step 2
  当前下游对象slot = 新下游对象ptr
}

//场景
A.添加下游对象(B, nil)   //A对象，删除B对象的引用。B被A删除，被标记为灰(如果B之前为白)
A.添加下游对象(B, C)     //A对象，更换下游B变成C。B被A删除，被标记为灰(如果B之前为白)
```

> 避免条件2（从灰色对象出发，到达白色对象的、未经访问过的路径被赋值器破坏），因为被删除对象，如果自身是灰色或者白色，则被标记为灰色

![delete_barrier_1](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/delete_barrier_1.png)

![delete_barrier_2](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/delete_barrier_2.png)

![delete_barrier_3](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/delete_barrier_3.png)

![delete_barrier_4](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/delete_barrier_4.png)

![delete_barrier_5](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/delete_barrier_5.png)

![delete_barrier_6](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/delete_barrier_6.png)

![delete_barrier_7](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/delete_barrier_7.png)

特点：标记结束不需要STW，但是回收精度低，GC 开始时STW 扫描堆栈记录初始快照，保护开始时刻的所有存活对象；且容易产生“冗余”扫描；

#### 混合屏障

大大缩短了 STW 时间

- GC 开始将栈上的对象全部扫描并标记为黑色；
- GC 期间，任何在栈上创建的新对象，均为黑色；
- 被删除的堆对象标记为灰色；
- 被添加的堆对象标记为灰色；
```
// 混合写屏障
func HybridWritePointerSimple(slot *unsafe.Pointer, ptr unsafe.Pointer) {
    shade(*slot)
    shade(ptr)
    *slot = ptr
}
```

### Golang GC过程

#### 标记清理

##### Marking setup

为了打开写屏障，必须停止每个goroutine，让垃圾收集器观察并等待每个goroutine进行函数调用， 等待函数调用是为了保证goroutine停止时处于安全点。

![stw_mark](https://liangyaopei.github.io/2021/01/02/golang-gc-intro/stw_mark.png)

```
// 如果goroutine4 处于如下循环中，运行时间取决于slice numbers的大小
func add(numbers []int) int {
    var v int
    for _, n := range numbers {
             v += n
     }
     return v
}
```

下面的代码中，由于 `for{}` 循环所在的goroutine 永远不会中断，导致始终无法进入STW阶段，资源浪费；Go 1.14 之后，此类goroutine 能被异步抢占，使得进入STW的时间不会超过抢占信号触发的周期，程序也不会因为仅仅等待一个goroutine的停止而停顿在进入STW之前的操作上。

```
func main() {
    go func() {
        for {
        }
    }()
    time.Sleep(time.Milliecond)
    runtime.GC()
    println("done")
}
```

##### Marking

一旦写屏障打开，垃圾收集器就开始标记阶段，垃圾收集器所做的第一件事是占用25%CPU。

标记阶段需要标记在堆内存中仍然在使用中的值。首先检查所有现goroutine的堆栈，以找到堆内存的根指针。然后收集器必须从那些根指针遍历堆内存图，标记可以回收的内存。

当存在新的内存分配时，会暂停分配内存过快的那些 goroutine，并将其转去执行一些辅助标记（Mark Assist）的工作，从而达到放缓继续分配、辅助 GC 的标记工作的目的。

##### Mark终止

关闭写屏障，执行各种清理任务（STW - optional ）

##### Sweep (清理)

清理阶段用于回收标记阶段中标记出来的可回收内存。当应用程序goroutine尝试在堆内存中分配新内存时，会触发该操作，清理导致的延迟和吞吐量降低被分散到每次内存分配时。

| 阶段 | 说明 | 赋值器状态 |
| --- | --- | --- |
| SweepTermination | 清扫终止阶段，为下一阶段的并发标记做准备工作，启动写屏障 | STW |
| Mark | 扫描标记阶段，与赋值器并发执行，写屏障开启 | 并发 |
| MarkTermination | 标记终止阶段，保证一个周期内标记任务完成，停止写屏障 | STW |
| GCoff | 内存清扫阶段，将需要回收的内存归还到堆中，写屏障关闭 | 并发 |
| GCoff | 内存归还阶段，将需要回收的内存归还给操作系统，写屏障关闭 | 并发 |

**清除阶段出现新对象：**

清除阶段是扫描整个堆内存，可以知道当前清除到什么位置，创建的新对象判定下，如果新对象的指针位置已经被扫描过了，那么就不用作任何操作，不会被误清除，如果在当前扫描的位置的后面，把该对象的颜色标记为黑色，这样就不会被误清除了

**什么时候进行清理？**

主动触发（runtime.GC()） 被动触发 （GC百分比、定时）

### GC 百分比

运行时中有GC 百分比的配置选项，默认情况下为100。此值表示在下一次垃圾收集必须启动之前可以分配多少新内存的比率。将GC百分比设置为100意味着：基于在垃圾收集完成后标记为活动的堆内存量，下次垃圾收集前，堆内存使用可以增加100%。

### GC过程演示

```
package main

import (
    "os"
    "runtime"
    "runtime/trace"
)

func gcfinished() *int {
    p := 1
    runtime.SetFinalizer(&p, func(_ *int) {
        println("gc finished")
    })
    return &p
}
func allocate() {
    _ = make([]byte, int((1<<20)*0.25))
}
func main() {
    f, _ := os.Create("trace.out")
    defer f.Close()
    trace.Start(f)
    defer trace.Stop()
    gcfinished()
    // 当完成 GC 时停止分配
    for n := 1; n < 50; n++ {
        println("#allocate: ", n)
        allocate()
    }
    println("terminate")
}
```

运行程序

```
liangyaopei> 
> $ GODEBUG=gctrace=1 go run main.go                                                                       
gc 1 @0.005s 3%: 0.023+0.87+0.059 ms clock, 0.19+0.80/0.42/0+0.47 ms cpu, 4->4->0 MB, 5 MB goal, 8 P
```

栈分析

```
gc 1      : 第一个GC周期
@0.005s   : 从程序开始运行到第一次GC时间为0.001 秒
5%        : 此次GC过程中CPU 占用率

wall clock
0.023+0.87+0.059 ms clock
0.023 ms  : STW，Marking Start, 开启写屏障
0.87 ms   : Marking阶段
0.059 ms  : STW，Marking终止，关闭写屏障

CPU time
0.19+0.80/0.42/0+0.47 ms cpu
0.19 ms   : STW，Marking Start
0.80 ms  : 辅助标记时间
0.42 ms  : 并发标记时间
0 ms   : GC 空闲时间
0.47 ms   : Mark 终止时间

4->4->0 MB， 5 MB goal
4 MB      ：标记开始时，堆大小实际值
4 MB      ：标记结束时，堆大小实际值
0 MB      ：标记结束时，标记为存活对象大小
5 MB      ：标记结束时，堆大小预测值

8 P
8P       ：本次GC过程中使用的goroutine 数量
```

## 关注指标与调优示例

### 关注指标

Go 的 GC 被设计为成比例触发、大部分工作与赋值器并发、不分代、无内存移动且会主动向操作系统归还申请的内存。因此最主要关注的、能够影响赋值器的性能指标有：

- CPU 利用率：回收算法会在多大程度上拖慢程序？有时候，这个是通过回收占用的 CPU 时间与其它 CPU 时间的百分比来描述的。
- GC 停顿时间：回收器会造成多长时间的停顿？目前的 GC 中需要考虑 STW 和 Mark Assist 两个部分可能造成的停顿。
- GC 停顿频率：回收器造成的停顿频率是怎样的？目前的 GC 中需要考虑 STW 和 Mark Assist 两个部分可能造成的停顿。
- GC 可扩展性：当堆内存变大时，垃圾回收器的性能如何？但大部分的程序可能并不一定关心这个问题。

### 调优示例

#### 合理化内存分配的速度、提高赋值器的 CPU 利用率

goroutine 的执行时间占其生命周期总时间非常短的一部分，但大部分时间都花费在调度器的等待上了，说明同时创建大量 goroutine 对调度器产生的压力确实不小，我们不妨将这一产生速率减慢，一批一批地创建 goroutine。

```
func concat() {
   for n := 0; n < 100; n++ {
      for i := 0; i < 8; i++ {
         go func() {
            s := "Go GC"
            s += " " + "Hello"
            s += " " + "World"
            _ = s
         }()
      }
   }
}

//改进
func concat() {
   wg := sync.WaitGroup{}
   for n := 0; n < 100; n++ {
      wg.Add(8)
      for i := 0; i < 8; i++ {
         go func() {
            s := "Go GC"
            s += " " + "Hello"
            s += " " + "World"
            _ = s
            wg.Done()
         }()
      }
      wg.Wait()
   }
}
```

#### 降低并复用已经申请的内存

`newBuf()` 产生的申请的内存过多, sync.Pool 是内存复用的一个最为显著的例子

```
func newBuf() []byte {
   return make([]byte, 10<<20)
}
b := newBuf()

//改进
var bufPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 10<<20)
    },
}
b := bufPool.Get().([]byte)
```

#### 调整 GOGC

降低收集器的启动频率（提高GC百分比）无法帮助垃圾收集器更快完成收集工作。降低频率会导致垃圾收集器在收集期间完成更多的工作。 可以通过减少新分配对象数量来帮助垃圾收集器更快完成收集工作

> 我的公众号：lyp分享的地方
> 
> 我的知乎专栏: [https://zhuanlan.zhihu.com/c\_1275466546035740672](https://zhuanlan.zhihu.com/c_1275466546035740672)
> 
> 我的博客： [www.liangyaopei.com](http://www.liangyaopei.com/)
> 
> Github Page: [https://liangyaopei.github.io/](https://liangyaopei.github.io/)