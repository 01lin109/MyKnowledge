# 1 .结合 AOP 与反射自动填充
在公司的自研框架中，你仿照事务控制，通过 AOP 拦截 `update/save` 类方法，并利用反射自动填充 `updateTime` 等字段，消除了上千行冗余代码 。

- **问题：** 请简述 Spring AOP 底层动态代理的两种实现机制（JDK 动态代理与 CGLIB）及它们的区别。如果在你的业务代码中，发生了**同类内部的方法调用**（例如 Service 类中的 `methodA` 直接调用了内部的 `save` 方法，且 `save` 方法上加了你的拦截注解），你的 AOP 拦截还能成功触发自动填充吗？为什么？如果不能，怎么解决？
## A.我的回答：
1.Spring AOP两种动态代理：
JDK动态代理的实现是基于接口的，只能对实现了接口的类进行动态代理，CGLIB的实现是通过继承，对指定的类生成一个子类并覆盖其中的方法。同类内部的方法调用这个时候AOP就无法成功拦截了，因为执行过程中调用methodA的本身就是由Spring创建的一个代理类，如何解决：还不知道
### 我的回答的问题：
- **关于失效的原因描述略显模糊**：你提到“调用 methodA 的本身就是由 Spring 创建的一个代理类”，这句是对的，但没解释清楚为什么 `methodA` 调 `save` 会失效。关键点在于：**`methodA` 内部调用 `save` 时，使用的是 `this` 关键字，而 `this` 指向的是原始的目标对象，而不是代理对象。**
    
- **解决方案缺失**：这是这道题的考点，通常有三种主流解法。

## B.标准回答：
### (1) Spring AOP 动态代理机制及区别

|**特性**|**JDK 动态代理**|**CGLIB 动态代理**|
|---|---|---|
|**实现原理**|基于 **Java 接口**。通过 `Proxy.newProxyInstance()` 创建。|基于 **继承（子类）**。通过 ASM 字节码技术生成目标类的子类。|
|**限制条件**|目标类 **必须实现至少一个接口**。|目标类不需要实现接口，但类和方法 **不能被 `final` 修饰**。|
|**Spring 选择策略**|如果目标对象实现了接口，默认使用 JDK 代理。|如果目标对象没有实现接口，则强制使用 CGLIB。|

---

### (2) 同类调用 AOP 拦截失效的原因
**拦截不能成功触发。**
**原理分析**：
AOP 的本质是通过**代理对象**在调用目标方法前后织入逻辑（比如反射填充字段）。
- 当你从外部调用 `service.methodA()` 时，调用的是 **Proxy（代理对象）**，拦截生效。
- 但是在 `methodA` 内部调用 `this.save()` 时，这里的 `this` 是 **Target（原始目标对象）**。由于原始对象的方法内部没有代理逻辑，因此 AOP 拦截器（Interceptor）根本不会被触发。

---
### (3) 解决方案
针对你的自研框架场景，有以下常见解法：
#### 方法一：通过 `AopContext` 获取当前代理对象（最常用）
这是 Spring 官方提供的“自救”方案。
1. 在启动类或配置类开启：`@EnableAspectJAutoProxy(exposeProxy = true)`。
2. 在代码中使用 `AopContext.currentProxy()` 强转为当前接口/类再进行调用：
    ```java
    public void methodA() {
        // 这样调用 save，本质上是代理对象在调用，拦截会生效
        ((YourService) AopContext.currentProxy()).save();
    }
    ```
#### 方法二：利用 Spring 注入自己（Self-Injection）
在 Service 类中通过 `@Autowired` 或 `@Resource` 注入自己。
```java
@Service
public class MyService {

    @Autowired
    private MyService self; // 关键：Spring 会将代理后的对象注入进来

    public void methodA() {
        // 不要直接调用 save()，那样用的是 this，拦截会失效
        // 使用注入的 self 调用，走的是代理对象，AOP 拦截成功
        self.save(); 
    }

    @AutoFill // 你的自研拦截注解
    public void save() {
        // 执行保存逻辑
    }
}
```

# 2.结合JVM内存模型优化导入模块
## A.我的回答：
1.传统POI全量加载：会迅速在堆的新生代中创建大量的对象，超出了堆的限制导致OOM
2.G1会维护一个优先队列，根据设定的每次收集时间有限选择回收价值最大的Region，而这样短命但巨大，或存活稍长的对象都属于回收价值很大的Region，从而使得内存能够得到及时释放防止OOM
### 🧐 回答中的瑕疵指正

1. **关于 POI OOM 的描述（细节不足）**：
    
    - 不仅仅是“对象很大直接进老年代”。POI 的问题在于其 **DOM 模型（User Model）**。它将 Excel 的每一行、每个单元格都封装成对象（如 `XSSFCell`），并维护复杂的树状结构和样式索引 。5k 行数据在 XML 解压后，内存膨胀倍数可达 10~20 倍。
        
2. **关于 G1 优先队列（术语偏差）**：
    
    - G1 确实有优先级，但它的核心术语是 **“Stop-The-World (STW) 可预测模型”**。
        
    - 对于“巨大对象”，G1 有专门的 **Humongous Region**，它的回收逻辑与普通 Region 不同，通常在 **Concurrent Marking（并发标记）** 阶段或 Full GC 时处理。
        

---

## B. 🏆 大厂满分复盘示例

你可以参考以下表达方式，展现出你不仅懂业务（EasyExcel），还深谙底层原理。

#### 1. 解析 POI 的 OOM 根源

在清能互联的项目中，原方案使用 POI 全量加载 5k+ 行数据。由于 POI 采用 **DOM 解析模式**，它会将整个 Excel 文档在内存中构建成一棵完整的对象树 。

- **内存放大效应**：即便原始文件仅几 MB，解压并转化成 `XSSFWorkbook` 后，由于海量的单元格对象（Cell）、样式对象（Style）以及复杂的引用关系，会迅速填满**新生代的 Eden 区** 。
    
- **晋升与溢出**：大量对象因无法被 Young GC 及时回收而晋升到**老年代**。当老年代也无法容纳这些密集的实例，且 Full GC 无法有效释放空间时，便抛出 `OutOfMemoryError`。
    

#### 2. G1 垃圾回收器对不同对象的处理策略

引入 **EasyExcel 流式解析**后，对象转为“随用随销” 。但在复杂业务（如设备名称映射）中，仍会有对象进入 G1 的监控视野。

- **处理“短命但巨大”的对象（Humongous Objects）**：
    
    - 在 G1 中，如果一个对象超过 Region 容量的一半，它会被判定为 **Humongous 对象**。
        
    - **物理存储**：G1 会分配一组**连续的 Region** 来存放它。
    - **回收时机**：是直接分配在老年代相关的区域。G1 会在 **Cleanup 阶段**或并发标记阶段发现该对象不再被引用时，立即对其回收，极大地降低了内存压力。
- **处理“存活稍长”的对象（Mixed GC）**：
    
    - 对于导入过程中因逻辑校验、转换而存活稍久的对象，它们会正常晋升到 **Old Region**。
        
    - **回收策略**：G1 维护一个 **CSet（回收集）**。它会根据用户设定的 `-XX:MaxGCPauseMillis`（预期停顿时间），优先选择那些垃圾占比最高（即回收价值最大）的 Old Region 进行回收。这种分阶段、按需回收的机制，保证了即便在处理大规模数据导入时，系统依然能保持极低的响应延迟。

# 3.反射性能优化
**标准回答：**

> **Spring AOP 底层机制：** Spring AOP 底层主要使用 **JDK 动态代理** 和 **CGLIB**。
> 
> - JDK 动态代理是基于接口的，利用 `InvocationHandler` 和反射机制生成实现接口的代理类。
>     
> - CGLIB 是基于类的，底层采用 ASM 字节码技术生成目标类的子类来重写方法。因此 CGLIB 不能代理 `final` 类和方法。Spring Boot 2.x 之后默认强制使用 CGLIB 构建代理。
>     
> 
> **反射性能优化手段（在自动审计组件中的应用）：**
> 
> 1. **缓存（Caching）：** 反射最耗时的步骤在于查找类的方法和属性（如 `getDeclaredMethod`）。我会在组件初始化时，针对涉及到的 Service 方法或实体类，将解析出来的 `Method` 或 `Field` 对象缓存到 `ConcurrentHashMap` 中，后续直接复用，避免重复查找。
>     
> 2. **关闭安全检查：** 在获取到 `Method` 或 `Field` 后，通过调用 `setAccessible(true)` 可以跳过 Java 的访问控制检查，这能带来显著的性能提升。
>     
> 3. **替代方案引入：** 如果性能要求极高，我会考虑放弃原生反射，引入高性能的字节码框架（如 `ReflectASM`）或者使用 `MethodHandles`（Java 7 引入的轻量级方法指针），它们在底层使用字节码生成，执行速度几乎等同于直接调用代码。



这里为你将本次复习的所有核心考点进行了系统化整理，按照知识库标准笔记格式（核心原理 + 关键代码/命令 + 面试标准答题话术）编排，你可以直接复制同步到你的 Obsidian / 知识库中：

---

# 📚 后端核心八股复习笔记（并发·AOP·算法·网络·数据库·Redis）

---

## 一、Java 并发：ConcurrentHashMap 核心机制

### 1. 底层结构与线程安全实现（JDK 8）

* **底层数据结构**：数组 + 单向链表 + 红黑树（链表长度 $\ge 8$ 且数组容量 $\ge 64$ 时树化，否则优先扩容）。
* **写操作并发控制**：
* **桶为空**：通过 `Unsafe.compareAndSwapObject`（CAS）无锁写入头节点。若失败则自旋重试。
* **桶非空**：使用 `synchronized` 锁住该哈希桶的**头节点**（Node / TreeNode），实现细粒度的槽位锁，不同桶写操作互不干扰。


* **协同扩容机制**：
* 扩容时通过 `sizeCtl` 记录扩容戳与并发线程数。
* 线程按步长（Stride，默认最小 16）领取数据迁移任务。
* 迁移完毕的桶头节点置为 `ForwardingNode`（hash = -1，带 `MOVED` 标志）。
* **读遇到 `ForwardingNode**`：通过其内部持有的 `nextTable` 无锁转发读取。
* **写遇到 `ForwardingNode**`：当前线程主动调用 `helpTransfer()` 协助迁移，迁移完成后再写入新表。



### 2. 复合操作原子性与累加方案

* **为什么 `get` + `put` 不安全**：`get()` 与 `put()` 单方法线程安全，但组合起来属于 Check-Then-Act 复合操作，存在脏读与更新丢失（Lost Update）。
* **原子更新方案**：
1. **原生函数式 API（推荐）**：
```java
// 计数统计原子累加
map.merge(key, 1L, Long::sum);
// 或
map.compute(key, (k, v) -> (v == null) ? 1L : v + 1L);

```


2. **结合 `LongAdder`（极高并发写推荐）**：
```java
map.computeIfAbsent(key, k -> new LongAdder()).increment();

```





---

## 二、Spring AOP 核心机制与工程陷阱

### 1. JDK 动态代理 vs CGLIB

| 维度 | JDK 动态代理 | CGLIB 动态代理 |
| --- | --- | --- |
| **底层原理** | 基于 Java 反射与 `java.lang.reflect.Proxy` 动态生成接口实现类 | 基于 ASM 字节码框架，在运行时生成目标类的子类 |
| **目标类要求** | 目标类**必须实现接口**，只能代理接口中声明的方法 | **无需接口**；但不能代理 `final` 类或 `final`/`private` 方法 |
| **Spring Boot 默认策略** | 仅在显式配置时启用 | **Spring Boot 2.x 起默认全量使用 CGLIB**（`spring.aop.proxy-target-class=true`） |

### 2. 类内部自调用 AOP 失效原因与解决方案

* **失效根本原因**：`methodA()` 内部直接调用 `methodB()` 本质上是 `this.methodB()`，`this` 指向的是**目标对象自身**而非外层的 Proxy 代理对象，直接绕过了切面拦截链。
* **解决方案**：
1. **自我注入（推荐）**：类内部 `@Autowired` 注入自身 Bean 引用（若有循环依赖告警配 `@Lazy`），通过 `self.methodB()` 调用。
2. **暴露代理对象**：配置类加 `@EnableAspectJAutoProxy(exposeProxy = true)`，通过 `((YourService) AopContext.currentProxy()).methodB()` 调用。
3. **架构重构**：将内部方法抽离到独立的 Service/组件中，通过外部依赖注入调用。



---

## 三、海量数据判重与分流：布隆 vs 布谷鸟过滤器

### 1. 布隆过滤器（Bloom Filter）

* **底层原理**：大型二进制位数组（BitMap） + $k$ 个单向 Hash 函数。写入时置 $k$ 个位为 1；查询时只要有一位为 0 则**绝对不存在**，全为 1 则**可能存在**。
* **假阳性（误判）**：说存在但实际不存在。
* **导入分流兜底方案**：
* **方案 A（多级校验）**：布隆过滤器粗筛拦掉绝大部分新增数据，流入更新流的极少数数据再做内存 Set 校验或查库精筛。
* **方案 B（DB 层面兜底）**：MySQL `INSERT INTO ... ON DUPLICATE KEY UPDATE`，消除分流判断错误带来的事务回滚。


* **致命缺陷**：**不支持物理删除**（多个元素共享 Bit 位，置 0 会破坏其他元素的映射）。

### 2. 布谷鸟过滤器（Cuckoo Filter，进阶方案）

* **核心结构**：哈希桶数组，每个桶含固定数量槽位（通常 4 个），存储元素的**指纹（Fingerprint，如 1~2 字节短哈希）**。
* **偏置双重哈希定位**：

$$i_1 = \text{Hash}(x)$$


$$i_2 = i_1 \oplus \text{Hash}(\text{fingerprint}(x))$$



*具有异或对称性：仅凭当前桶索引与指纹，即可算出备选桶位置，无需原始数据。*
* **三大优势**：
1. **原生支持动态物理删除**（直接在槽位比对并清空指纹）。
2. **低误报率下空间利用率明显超越布隆过滤器**。
3. **CPU 缓存友好**（最多只读写两个桶的连续内存，而布隆需离散跳跃访问 $k$ 个位）。



---

## 四、网络通信与 RPC 底层（Feign · 连接池 · TCP）

### 1. Feign 默认客户端与连接池

* **默认底层**：JDK 自带的 `HttpURLConnection`（无连接池，高并发下频繁进行 TCP 握手挥手短连接，导致 CPU 飙高、RT 激增）。
* **优化策略**：替换为支持连接复用与长连接的 **Apache HttpClient** 或 **OkHttp**。
* **连接池关键参数**：
* `max-total`：客户端全局最大物理连接数。
* `default-max-per-route`：每个具体路由/目标域名的最大并发连接数（极为关键，默认通常仅 2~5，未调大会导致并发请求大量排队阻塞）。



### 2. TCP 挥手与故障排查

* **`TIME_WAIT`（主动关闭方进入）**：
* **时长**：必须等待 2MSL（报文最大生存时间）。
* **核心作用**：
1. 确保最后的 ACK 能到达对方（若丢失，对方重发 FIN，主动方能重新发 ACK 并正常关闭）。
2. 确保网络中残余的孤儿报文彻底消亡，避免混入复用同端口的新连接。


* **大量出现的原因**：本机主动关闭了大量连接（如高频 HTTP 短连接未开启 Keep-Alive），容易导致可用端口耗尽。


* **`CLOSE_WAIT`（被动关闭方进入）**：
* **核心原因**：对方已发送 FIN 要求关闭，但**我方代码没有调用 `close()` 关闭连接/流**。
* **常见排查点**：HTTP 响应体、文件输入输出流在发生异常后**未在 `finally` 块中显式关闭**。
* **线上灾难**：耗尽系统的文件句柄数（File Descriptor），引发 `java.io.IOException: Too many open files` 导致进程瘫痪。



---

## 五、MySQL InnoDB 事务隔离与锁机制

### 1. 隔离级别与幻读解决机制

* **默认隔离级别**：**可重复读（Repeatable Read, RR）**。
* **快照读（普通 SELECT）**：基于 **MVCC（多版本并发控制）+ ReadView 一致性视图 + undo log 版本链** 解决幻读（始终按快照版本读取）。
* **当前读（SELECT FOR UPDATE / UPDATE / DELETE / INSERT）**：基于 **Next-Key Lock（记录锁 Record Lock + 间隙锁 Gap Lock）** 解决幻读（锁住记录及前后开区间间隙，阻止其他事务向间隙插入新记录）。

### 2. 行锁基本单位与无索引 UPDATE 的后果

* **行锁基本单位**：**Next-Key Lock（左开右闭区间）**。命中唯一索引等值查询且记录存在时降级为 Record Lock，记录不存在时退化为 Gap Lock。
* **无索引 UPDATE 的灾难**：
* InnoDB 的行锁本质是**加在索引项上的锁**。
* 若 `WHERE` 条件无索引或索引失效，引擎将走**主键聚簇索引的全表扫描**，导致**整张表的所有记录与全部间隙被通通加上 Next-Key Lock**（相当于锁全表），其他事务的所有写操作陷入长时间阻塞与死锁。



---

## 六、Redis 性能原理、高可用与分布式锁

### 1. Redis 高性能四大支柱

1. **纯内存操作**：无磁盘随机寻道损耗。
2. **单线程主模型**：核心执行线程单一，避免 CPU 上下文切换、死锁与并发竞争。
3. **I/O 多路复用**：基于 Reactor 模式与 `epoll` 事件驱动，单线程高吞吐处理海量 Socket 读写。
4. **高效数据结构与编码优化**：SDS 杜绝溢出且 $O(1)$ 获取长度；小数据量采用 Listpack/QuickList 连续内存紧凑排布。

### 2. 缓存异常三兄弟极简速记

* **缓存穿透**（DB 和缓存均无数据）：
* *防御*：布隆/布谷鸟过滤器前置拦截非法 Key；未命中时回写短期空值（Cache Null）。


* **缓存击穿**（单个超热 Key 到期）：
* *防御*：分布式互斥锁（Mutex）控制单线程查库回填；或逻辑不过期（后台异步刷新）。


* **缓存雪崩**（大批 Key 集中过期或 Redis 宕机）：
* *防御*：设置 TTL 追加随机扰动时间（1~5 分钟）；部署 Redis 集群高可用；结合本地缓存（Caffeine）与限流降级。



### 3. Redis 分布式锁生产实践

* **`SETNX` + `EXPIRE` 分步缺陷**：非原子操作。中途进程崩溃会导致 Key 永远无过期时间，造成永久死锁。
* **标准原子加锁命令**：
```bash
SET lock_key unique_token NX PX 30000

```


* **业务执行超时问题与看门狗（Watchdog）机制**：
* 业务超时会导致锁被 Redis 自动删除，引发并发竞态及原线程误删他人锁的问题。
* **解法（以 Redisson 为例）**：加锁成功后开启后台定时守护任务（默认每隔 10s 即 TTL/3），若业务线程未完成则自动延长过期时间；业务结束或宕机后守护线程终止，锁自然释放。

### JVM 垃圾回收与内存分配

**1. 对象生死判断**

* **可达性分析**：以 GC Roots 为起点向下搜索引用链，不可达即标记为可回收。
* **GC Roots 范畴**：虚拟机栈局部变量、本地方法栈（JNI）引用、方法区静态属性与常量、`synchronized` 持有的对象锁等。

**2. 核心回收算法**

* **标记-清除**：简单但产生大量内存碎片；老年代（早期 CMS）。
* **标记-复制**：无碎片、指针碰撞分配快，但存在空间折损；新生代（Eden:Survivor=8:1:1）。
* **标记-整理**：无碎片、无空间浪费，但移动存活对象需要 STW 修正指针；老年代（Parallel Old）。

**3. 内存分配与晋升机制**

* **分配路径**：优先在线程私有的 TLAB 分配 $\to$ 空间不足进 Eden $\to$ 超大对象（`-XX:PretenureSizeThreshold`）直接进老年代。
* **晋升老年代条件**：
* 达到年龄阈值（`-XX:MaxTenuringThreshold`，默认 15）。
* 动态年龄判定：Survivor 中同龄对象大小总和超 Survivor 空间 50%，$\ge$ 该年龄的对象直升。
* 分配担保：Young GC 后存活对象超过 Survivor 承载能力，溢出部分移入老年代。



**4. GC 触发时机**

* **Young GC**：Eden 区空间耗尽。
* **Full GC**：老年代空间不足、元空间耗尽、调用 `System.gc()`、老年代空间分配担保失败、CMS 并发模式失败（Concurrent Mode Failure）。

---

### CMS / G1 / ZGC 演进与三色标记

**1. 收集器演进对比**

* **CMS**：以降低停顿为目标的老年代收集器。流程：初始标记(STW) $\to$ 并发标记 $\to$ 重新标记(STW) $\to$ 并发清除。缺点：碎片多、浮动垃圾、易并发失败退化为 Serial Old。
* **G1**：面向整堆的 Region 化架构。流程：初始标记(STW) $\to$ 并发标记 $\to$ 最终标记(STW) $\to$ 筛选回收(STW)。局部复制、整体整理，通过停顿预测模型优先回收高价值 Region。
* **ZGC**：面向超大堆的亚毫秒级低延迟收集器。核心技术为**染色指针（44位地址+4位GC标志）**与**读屏障自愈（Load Barrier / Self-Healing）**，将对象重定位完全并发化，停顿与堆大小无关。

**2. G1 的两个 Set**

* **RSet（Remembered Set，记忆集）**：“谁引用了我”。记录外部 Region 到当前 Region 的跨代/跨区引用（Points-into），作为额外 GC Roots，避免全堆扫描。
* **CSet（Collection Set，回收集）**：“本次回收谁”。根据预期停顿时间挑选收益最高的分区集合，包含全部年轻代 Region + 部分老年代 Region。

**3. 三色标记与漏标**

* **颜色定义**：白色（未访问/垃圾）、灰色（本对象已访问但字段未扫描完）、黑色（本对象及字段均已扫描完）。
* **漏标发生条件**：黑色对象新增指向白色对象的引用，且灰色对象断开对该白色对象的直接/间接引用。
* **解决方案**：
* **CMS 增量更新（Incremental Update）**：写后屏障。黑色插入白色引用后将黑色标灰（记录脏卡），Remark 阶段重新深度扫描。
* **G1 原始快照（SATB）**：写前屏障。断开引用前将旧引用推入 SATB 队列标灰，不重复深扫，保留快照，容忍当期浮动垃圾。



---

### Java 并发核心

**1. `volatile` 关键字**

* **核心语义**：保证**可见性**（写后立即刷回主存，读前强制主存重载）、保证**有序性**（禁止指令重排）；**不保证原子性**（如 `count++` 不安全）。
* **底层落地**：JMM 插入 4 类内存屏障（`LoadLoad`、`StoreStore`、`LoadStore`、`StoreLoad`）；x86 汇编层面使用带 `lock` 前缀的指令（如 `lock addl`），强制清空写缓冲并触发 MESI 缓存一致性。
* **典型场景**：状态标志位；单例模式双重检查锁定（DCL）防止对象半初始化逸出。

**2. 线程池拒绝策略与风险**

* **4 种拒绝策略**：
* `AbortPolicy`（默认）：抛出 `RejectedExecutionException`。
* `CallerRunsPolicy`：调用者线程同步执行任务，兼具负反馈限流效果。
* `DiscardPolicy`：静默丢弃任务。
* `DiscardOldestPolicy`：丢弃队列头最老任务并重试提交。


* **Executors 禁用原因**：
* `FixedThreadPool` / `SingleThreadExecutor`：使用无界队列 `LinkedBlockingQueue`（容量 $2^{31}-1$），积压导致堆内存 OOM。
* `CachedThreadPool`：最大线程数无界（`maximumPoolSize = Integer.MAX_VALUE`），高并发下疯狂创建线程导致系统线程耗尽 OOM。



**3. `synchronized` 锁升级流程**

* **无锁（`0 01`）** $\to$ **偏向锁（`1 01`）**：CAS 记录线程 ID 到 Mark Word，后续同线程进入无开销。
* **轻量级锁（`00`）**：存在线程交替竞争时偏向锁撤销，栈帧分配 Lock Record，CAS 替换 Mark Word 为指向栈帧指针，失败触发自适应自旋。
* **重量级锁（`10`）**：自旋超限或多线程激烈竞争，Mark Word 指向底层 `ObjectMonitor`，线程挂起阻塞，依赖操作系统互斥原语，产生用户态/内核态切换开销。

**4. AQS 核心原理与应用**

* **三大组件**：`volatile int state`（状态计数器）、CLH 变体双向队列（排队阻塞）、模板方法（子类实现 `tryAcquire` / `tryRelease` 等）。
* **典型落地**：
* `ReentrantLock`：`state` 记录独占锁状态与重入次数。
* `Semaphore`：`state` 记录剩余可用许可证（共享）。
* `CountDownLatch`：`state` 记录倒计数剩余次数（共享）。
* `ReentrantReadWriteLock`：一个 32 位 `state` 高 16 位存读锁，低 16 位存写锁。



**5. ThreadLocal 结构与内存泄漏**

* **数据结构**：每个线程持有独立 `ThreadLocalMap`，底层为线性探测法数组，`Entry` 的 Key 继承 `WeakReference<ThreadLocal<?>>`，Value 为强引用的业务对象。
* **Key 的独立性**：同一个线程内，不同的 `ThreadLocal` 实例作为完全独立的 Key 存放在 Map 数组的不同槽位，不会冲突覆盖。
* **泄漏本质**：Key 为弱引用在 GC 时被回收变为 `null`，但在线程池环境中线程长期存活，强引用链 `Thread -> ThreadLocalMap -> Entry -> Value` 无法断开，导致 Value 悬空泄漏。
* **防御规范**：使用完后必须在 `finally` 块中显式执行 `threadLocal.remove()`。