# 1.ConCurrentHashMap

#### **第一部分：JDK 8 中 `ConcurrentHashMap` 的底层结构与并发控制机制**

1. **底层数据结构**：
      
    - JDK 8 的 `ConcurrentHashMap` 废弃了 JDK 7 的 Segment 分段锁机制，底层结构与同版本的 `HashMap` 保持一致，采用 **“数组 + 单向链表 + 红黑树”**。

    - 当链表长度达到阈值（默认为 8）且数组容量大于等于 64 时，链表会树化为红黑树；若容量小于 64 则优先扩容。
        
2. **写操作并发保证（CAS + synchronized）**：
    - **节点为空时**：利用 `Unsafe` 的 **CAS** 操作无锁尝试插入头节点。如果 CAS 成功则写入完毕；若失败（说明有其他线程竞争），则自旋重试进入下一轮逻辑。
    - **节点不为空时**：使用 **`synchronized` 锁住该哈希桶的头节点（Node/TreeNode）**，实现细粒度的锁化。锁的粒度仅针对当前哈希槽位，不同桶之间的写操作互不阻塞。
3. **并发扩容与数据迁移机制（多线程协作扩容）**：
    - **并发协作**：扩容时通过 `sizeCtl` 变量记录扩容戳与参与扩容的线程数。支持多线程协同完成数据迁移，每个线程通过 CAS 领取一段区间（Stride，默认最小 16 个桶）的任务。
    - **ForwardingNode 标记**：已迁移完毕的桶，头节点会被替换为特定的 `ForwardingNode`（其 hash 值为 -1，带有 `MOVED` 标志）。
    - **读写处理**：
        - **读操作**遇到 `ForwardingNode` 时，会通过其内部持有的 `nextTable` 引用转发到新表中无锁读取，读操作全程不阻塞。
        - **写操作**遇到 `ForwardingNode` 时，当前线程会主动调用 `helpTransfer()` 协助扩容，迁移完成后再在新表执行写入。
#### **第二部分：复合操作的线程安全陷阱与原子更新方案**

1. **为什么 `map.get()` + `map.put()` 不安全？**
    - `ConcurrentHashMap` 的单方法（如单次 `get` 或单次 `put`）虽然是线程安全的，但 **`get()` + 判断/计算 + `put()` 属于典型的“复合操作（Check-Then-Act）”**。
        
    - 在多线程并发执行下，多个线程可能同时通过 `get()` 读取到相同的旧值，在各自工作内存中计算后分别执行 `put()`，导致后执行的写覆盖先执行的写（Lost Update），破坏原子性。
        
2. **JDK 8 中的优雅原子更新方案**：
    
    - **推荐方案一：`compute()` / `computeIfPresent()` / `merge()`（Lambda 原生支持）**
        - 底层会对桶头节点加锁，保证“读取-计算-写回”全过程是**单次原子操作**。
        - 示例代码（计数累加）：
            ```java
            // merge: 键不存在设为1，已存在则执行累加
            map.merge(exceptionKey, 1L, Long::sum);
            
            // 或者使用 compute:
            map.compute(exceptionKey, (k, v) -> (v == null) ? 1L : v + 1L);
            ```
            
    - **推荐方案二：结合 `LongAdder` / `AtomicLong`**
        - Map 中存放引用类型计数器，通过 `computeIfAbsent` 确保计数器对象创建的原子性，再由原子类自身处理数值累加：
            
            ```
            map.computeIfAbsent(exceptionKey, k -> new LongAdder()).increment();
            ```
            
        - **优势**：在高并发写极端激烈的场景下，`LongAdder` 的分段计数（Cell 数组）机制性能优于对整个哈希槽锁定的反复 CAS/`synchronized`。