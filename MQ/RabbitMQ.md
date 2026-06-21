https://rabbitmq.cn/docs
![[7bf15d7f2e88d5e293023addf16d1eac.png]]
# 1.消息可靠性
![[Pasted image 20260516162207.png]]
## 1.1 发送者的可靠性

**发送者可靠性**指的是在消息从生产者发送到 RabbitMQ Broker 的过程中，确保消息不会因为网络、Broker 宕机或其他原因而丢失。

可靠性问题主要产生于以下原因：

### 1️⃣ 产生原因

1. **网络或Broker宕机**
    - 消息发送过程中，如果网络断开或者 RabbitMQ Broker 崩溃，未送达的消息可能丢失。
2. **发送模式不持久化**
    - 如果消息 `deliveryMode=1`（非持久化），即便 Broker 收到消息但还没写磁盘，也可能在 Broker 重启时丢失。
3. **批量发送未确认**
    - 生产者发送大量消息时，如果没有确认机制，无法保证每条消息都被 Broker 接收。

---

### 2️⃣ 生产者重连

当生产者与 RabbitMQ 连接断开时，生产者通常需要做 **自动重连**：

- **自动重连逻辑**：
    1. 检测到连接断开或发送异常。
    2. 按策略重试连接（指数退避、固定间隔等）。
    3. 重新声明必要的 Exchange、Queue 等资源。
- **注意事项**：
    - 在重连期间可能积压未发送的消息，需要在本地缓存（内存或持久化存储）消息。
    - 防止消息重复发送或丢失，需要与确认机制结合使用。

---

### 3️⃣ 生产者确认（Publisher Confirms）

**Publisher Confirms** 是 RabbitMQ 提供的一种可靠性机制，用于确认 Broker 是否成功接收消息：

1. **同步确认**
    - 发送一条消息，等待 Broker 返回 ACK/NACK 再继续发送。
    - 优点：可靠性最高，缺点：吞吐量低。
2. **异步确认**
    - 生产者发送消息后，不阻塞等待 ACK，而是注册回调函数处理确认。
    - 优点：吞吐量高，仍保证消息送达 Broker。
3. **未确认消息处理**
    - 若收到 NACK 或超时未确认，需要重新发送消息。
    - 通常结合本地缓存队列，避免消息丢失。

---

## 1.2MQ的可靠性
### 1️⃣ 数据持久化（Persistence）

**目标**：保证消息即使在 Broker 重启或宕机时也不丢失。

**核心点**：

1. **队列持久化 + 消息持久化**
    - 队列持久化（`durable=true`）：队列结构重启后仍在。
    - 消息持久化（`deliveryMode=2`）：消息写入磁盘，即便 Broker 崩溃也可恢复。
2. **持久化写入机制**
    - RabbitMQ 默认使用 **内存 + 磁盘** 的混合方式。
    - 消息先写入内存队列，然后异步刷到磁盘（持久化）。
    - 如果 Broker 在磁盘刷写完成前崩溃，部分持久化消息可能丢失。
    - 解决方案：
        - 使用 **Publisher Confirms** 确保 Broker 已成功接收消息。
        - 或开启 **事务模式**，虽然吞吐量下降。
3. **高可靠性策略**
    - **镜像队列（Mirrored Queue / HA Queue）**：
        - 队列和消息同步到多个节点，单节点宕机不会丢消息。
        - 配合持久化，提供企业级可靠性。

---

### 2️⃣ Lazy Queue（惰性队列）

**目标**：在消息量非常大时，减少内存占用和潜在消息丢失风险。

**核心点**：

1. **原理**
    - 普通队列会尽量把消息保存在内存里，提高吞吐量。
    - Lazy Queue 会尽量把消息 **写入磁盘**，只在消费时才加载到内存。
2. **优势**
    - **降低内存压力**：海量消息不会占用大量 RAM，减少 Broker 崩溃风险。
    - **提高可靠性**：磁盘存储比纯内存更可靠，消息丢失概率降低。
3. **适用场景**
    - 消息量大且消费者处理较慢的情况（如日志收集、任务队列）。
    - 不追求超高吞吐量，但追求可靠性。
4. **配置示例**（Java 客户端）：
    
    ```
    Map<String, Object> args = new HashMap<>();args.put("x-queue-mode", "lazy");channel.queueDeclare("myLazyQueue", true, false, false, args);
    ```
    
5. **注意事项**
    - Lazy Queue 吞吐量略低，因为每次消费都可能涉及磁盘 IO。
    - 可以结合 **消息持久化 + Publisher Confirms**，进一步确保可靠性。

---

### 3️⃣ 总结 RabbitMQ 可靠性策略

|维度|作用 / 特性|
|---|---|
|消息持久化|确保消息写磁盘，Broker 重启可恢复|
|队列持久化|队列结构存活，结合消息持久化保证消息不丢|
|Publisher Confirms|生产者确认消息已被 Broker 接收|
|Lazy Queue|减少内存压力，海量消息仍可靠|
|镜像队列 / HA Queue|多节点同步，单点宕机不丢消息|

✅ 核心原则：**“队列 + 消息持久化 + 适当的队列策略（Lazy/Mirrored） + 确认机制”**，是 RabbitMQ 保证可靠性的全套方案。
## 1.3 消费者的可靠性
消费者可靠性主要关注的是 **消息被安全、正确地消费**，尤其是在消费者宕机、网络断开或者处理异常时，如何保证消息不会丢失或重复。

---
### 1️⃣ 产生原因（为什么需要关注消费者可靠性）

1. **消费者宕机或网络异常**
    - 消费者取到消息但未处理完成就崩溃，可能导致消息丢失。
2. **消息处理失败**
    - 消费逻辑异常或外部依赖出错，需要重试机制。
3. **批量消费或异步消费**
    - 如果不确认消息，Broker 可能错误认为消息已被消费，从而丢失消息。
4. **流量过大导致阻塞**
    - 消费者处理速度慢，消息积压在 Broker，可能影响系统稳定性。

---

### 2️⃣ 消费者可靠性保障机制

#### 2.1 消费者确认（Acknowledgement, ACK）

- 消费者处理完消息后 **手动发送 ACK** 给 Broker，表示消息已安全消费。
- **方式**：
    ```java
    channel.basicConsume(queueName, false, consumer); // false 表示手动 ACK
    channel.basicAck(deliveryTag, false);
    ```
    
- **未确认消息处理**：
    - 如果消费者挂掉，Broker 会 **把未确认的消息重新放回队列**，保证消息不会丢失。
- **注意**：
    - 自动 ACK（`autoAck=true`）风险高：消息取到就认为消费成功，处理失败会丢失消息。

---

#### 2.2 消费者重试与死信队列（DLX, Dead Letter Exchange）

- 消费者处理异常时，消息可以：
    - **重新放回队列**（重新消费）
    - **投递到死信队列**（记录失败消息，人工或程序处理）
- 配置示例：
    
    ```java
    Map<String, Object> args = new HashMap<>();
    args.put("x-dead-letter-exchange", "dlx.exchange");
    channel.queueDeclare("normalQueue", true, false, false, args);
    ```
    

---

#### 2.3 流量控制（QoS / Prefetch）

- **目的**：防止消费者过载，保证消息可靠处理。
- RabbitMQ 支持 **prefetch** 限制一次消费的消息数：
    
    ```java
    channel.basicQos(10); // 每次最多取 10 条消息
    ```
    
- 好处：
    - 消费者处理慢时，消息不会在内存中积压太多。
    - 提高处理成功率，降低丢失或重复概率。

---

#### 2.4 幂等消费

- 即使消息被重复投递，也不会影响业务结果。
- 通常结合：
    - **消息唯一 ID**
    - **去重逻辑**
- 这在分布式系统中非常重要，保证消费者可靠性和数据一致性。

---

### 3️⃣ 总结 RabbitMQ 消费者可靠性策略

|机制|作用|
|---|---|
|**手动 ACK**|确认消息处理完成，未 ACK 消息可重投|
|**死信队列（DLX）**|处理消费失败的消息，避免丢失|
|**Prefetch / QoS**|控制消息流量，防止消费者过载|
|**幂等消费**|防止重复消费带来的数据错误|
|**重连机制**|消费者掉线后自动重连 Broker，继续消费|

✅ 核心原则：

## 1.4延迟消息

延迟消息是指 **生产者发送消息后，不立即投递给消费者，而是在指定时间后才被消费** 的消息。
应用场景：

- 定时任务（比如订单超时自动取消）
    
- 延时通知（比如 30 分钟后提醒用户）
    
- 重试机制（处理失败的消息延时重发）
    

---

### 1️⃣ RabbitMQ 原生延迟消息实现方式

RabbitMQ 本身并没有内置延迟队列机制，但可以通过 **死信队列（Dead Letter Exchange, DLX）** 和 **消息 TTL（Time-To-Live）** 实现。

---

### 2️⃣ 消息 TTL（Time-To-Live）

- **作用**：设置消息在队列中存活的时间。
    
- 配置方式：
    
    ```java
    Map<String, Object> args = new HashMap<>();
    args.put("x-message-ttl", 30000); // 30秒
    channel.queueDeclare("delayQueue", true, false, false, args);
    ```
    
- 消息在 TTL 到期后，如果没有被消费，会触发 **死信处理**。
    

---

### 3️⃣ 死信交换机（Dead Letter Exchange, DLX）

- **作用**：当消息被拒绝、过期（TTL）或队列满时，Broker 会将消息发送到 DLX。
    
- **结合延迟消息**：
    
    1. 创建一个延迟队列（TTL 队列），消息在此队列中等待指定时间。
        
    2. 配置 DLX，将过期的消息投递到实际消费队列。
        
- 配置示例：
    
    ```java
    Map<String, Object> args = new HashMap<>();
    args.put("x-dead-letter-exchange", "realExchange"); // 死信交换机
    args.put("x-message-ttl", 60000); // 60秒延迟
    channel.queueDeclare("delayQueue", true, false, false, args);
    ```
    

---

### 4️⃣ 延迟消息处理流程（DLX 方式）

1. 生产者发送消息到 **延迟队列**，设置 TTL。
    
2. 消息在延迟队列中等待 TTL 时间到期。
    
3. TTL 到期 → 队列触发死信机制 → 消息被发送到 **死信交换机**。
    
4. DLX 将消息路由到 **实际消费队列**。
    
5. 消费者从实际队列消费消息。
    

> 核心思想：**用 TTL + DLX 模拟延迟消费**。

---

### 5️⃣ RabbitMQ 延迟消息插件

- RabbitMQ 官方提供了 **rabbitmq_delayed_message_exchange** 插件，可直接支持延迟投递：
    
    - 不需要 TTL + DLX 组合，直接在 Exchange 发送消息时指定延迟时间。
        
    - 配置简单，性能稍优于 TTL + DLX 方法。
        
- 示例：
    
    ```java
    Map<String, Object> args = new HashMap<>();
    args.put("x-delayed-type", "direct");
    channel.exchangeDeclare("delayedExchange", "x-delayed-message", true, false, args);
    
    AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
        .headers(Collections.singletonMap("x-delay", 10000)) // 延迟 10 秒
        .build();
    channel.basicPublish("delayedExchange", "routingKey", props, messageBody.getBytes());
    ```
    

---

### 6️⃣ 总结延迟消息关键点

|概念|作用|
|---|---|
|**消息 TTL**|控制消息在队列中的存活时间|
|**死信交换机 DLX**|TTL 到期的消息路由到目标队列，实现延迟投递|
|**延迟队列**|队列结合 TTL + DLX 构建延迟消息队列|
|**插件方式（x-delayed-message）**|官方支持延迟消息交换机，简化实现|

✅ 核心原则：

> 延迟消息 = “先放一个等待队列（TTL）→ 到期触发死信 → 真正投递到消费者队列”，或者用延迟插件直接实现。

---

如果你愿意，我可以帮你画一张 **延迟消息流程图**，把 **生产者 → 延迟队列 → DLX → 实际队列 → 消费者** 全流程可视化，看图理解比文字直观。

你希望我画吗？