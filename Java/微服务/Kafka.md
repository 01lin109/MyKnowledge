# 一、Kafka 快速上手

> 作者：楼兰

这一章节主要是快速了解 Kafka 产品，搭建 Kafka 服务，并开始了解 Kafka 的基础功能。

---

## 一、快速了解 Kafka

### 1、MQ 的作用

MQ：MessageQueue，消息队列。队列，是一种 FIFO 先进先出的数据结构。消息则是跨进程传递的数据。一个典型的 MQ 系统，会将消息由生产者发送到 MQ 进行排队，然后根据一定的顺序交由消息的消费者进行处理。

QQ 和微信就是典型的 MQ。只不过它对接的使用对象是人，而 Kafka 需要对接的使用对象是应用程序。MQ 的作用主要有以下三个方面：

#### 异步

**例子**：快递员发快递，直接到客户家效率会很低。引入菜鸟驿站后，快递员只需要把快递放到菜鸟驿站，就可以继续发其他快递去了。客户再按自己的时间安排去菜鸟驿站取快递。

**作用**：异步能提高系统的响应速度、吞吐量。

#### 解耦

**例子**：《Thinking in JAVA》很经典，但是都是英文，我们看不懂，所以需要编辑社，将文章翻译成其他语言，这样就可以完成英语与其他语言的交流。

**作用**：

1. 服务之间进行解耦，才可以减少服务之间的影响。提高系统整体的稳定性以及可扩展性。
2. 另外，解耦后可以实现数据分发。生产者发送一个消息后，可以由一个或者多个消费者进行消费，并且消费者的增加或者减少对生产者没有影响。

#### 削峰

**例子**：长江每年都会涨水，但是下游出水口的速度是基本稳定的，所以会涨水。引入三峡大坝后，可以把水储存起来，下游慢慢排水。

**作用**：以稳定的系统资源应对突发的流量冲击。

### 2、Kafka 产品介绍

Kafka 是目前最具影响力的开源 MQ 产品，官网地址：https://kafka.apache.org/

Apache Kafka 最初由 LinkedIn 开发并于 2011 年开源。它主要解决大规模数据的实时流式处理和数据管道问题。Kafka 是一个分布式的发布-订阅消息系统，可以快速地处理高吞吐量的数据流，并将数据实时地分发到多个消费者中。Kafka 消息系统由多个 broker（服务器）组成，这些 broker 可以在多个数据中心之间分布式部署，以提供高可用性和容错性。

Kafka 使用高效的数据存储和管理技术，能够轻松地处理 TB 级别的数据量。其优点包括高吞吐量、低延迟、可扩展性、持久性和容错性等。

Kafka 在企业级应用中被广泛应用，包括实时流处理、日志聚合、监控和数据分析等方面。同时，Kafka 还可以与其他大数据工具集成，如 Hadoop、Spark 和 Storm 等，构建一个完整的数据处理生态系统。

### 3、Kafka 的特点

Kafka 最初诞生于 LinkedIn 公司，其核心作用就是用来收集并处理庞大复杂的应用日志。一个典型的日志聚合应用场景如下：

```
Application    nginx    log4j    ...
    ↓           ↓        ↓
Producer: Flume / LogStash
    ↓
Kafka  →  Kafka  →  Kafka
    ↓
Flink / Spark / Streams
    ↓
微服务链路追踪 / SEO统计 / 用户行为分析
    ↓
ES / Hadoop
```

业务场景决定了产品的特点。所以 Kafka 最典型的产品特点有以下几点：

1. **数据吞吐量很大**：需要能够快速收集各个渠道的海量日志。
2. **集群容错性高**：允许集群中少量节点崩溃。
3. **功能不需要太复杂**：Kafka 的设计目标是高吞吐、低延迟和可扩展，主要关注消息传递而不是消息处理。所以，Kafka 并没有支持死信队列、顺序消息等高级功能。
4. **允许少量数据丢失**：在海量的应用日志中，少量的日志丢失是不会影响结果的。所以 Kafka 的设计初衷是允许少量数据丢失的。当然 Kafka 本身也在不断优化数据安全问题。

---

## 二、快速上手 Kafka

### 1、快速搭建单机服务

Kafka 的运行环境非常简单，只要有 JVM 虚拟机就可以进行。这里，我们使用一台安装了 JDK 1.8 的 CentOS 9 机器作为演示。

JDK 的安装过程略。

**下载 Kafka**。官网下载地址：https://kafka.apache.org/downloads

这里我们选择下载 kafka 2.13-3.8.0.tgz。关于 Kafka 的版本，前面的 2.13 是开发 Kafka 的 Scala 语言的版本，后面的 3.8.0 是 Kafka 应用的版本。Scala 是一种运行于 JVM 虚拟机之上的语言。在运行时，只需要安装 JDK 就可以了，选哪个 Scala 版本没有区别。但是如果要调试源码，就必须选择对应的 Scala 版本。因为 Scala 语言的版本并不是向后兼容的。

**下载 Zookeeper**，下载地址 https://zookeeper.apache.org/releases.html ，Zookeeper 的版本并没有强制要求，这里我们选择 3.8.4 版本。

> Kafka 的安装程序中自带了 Zookeeper，可以在 Kafka 的安装包的 libs 目录下查看到 Zookeeper 的客户端 jar 包。但是，通常情况下，为了让应用更好维护，我们会使用单独部署的 Zookeeper，而不使用 Kafka 自带的 Zookeeper。

下载完成后，将这两个工具包上传到服务器上，解压后，分别放到 `/app/kafka` 和 `/app/zookeeper` 目录下。然后配置 KAFKA_HOME 环境变量指向 Kafka 安装目录。接下来将这两个组件部署目录下的 bin 目录路径配置到 PATH 环境变量中。

下载下来的 Kafka 安装包不需要做任何的配置，就可以直接单机运行。这通常是快速了解 Kafka 的第一步。

**启动 Kafka 之前需要先启动 Zookeeper**。这里就用 Kafka 自带的 Zookeeper。启动脚本在 bin 目录下。

```bash
cd $KAFKA_HOME
nohup bin/zookeeper-server-start.sh config/zookeeper.properties &
```

注意下脚本是不是有执行权限。

从 nohup.out 中可以看到 Zookeeper 默认会在 2181 端口启动。通过 jps 指令看到一个 QuorumPeerMain 进程，确定服务启动成功。

**启动 Kafka**：

```bash
nohup bin/kafka-server-start.sh config/server.properties &
```

启动完成后，使用 jps 指令，看到一个 Kafka 进程，确定服务启动成功。服务会默认在 9092 端口启动。

### 2、简单收发消息

Kafka 的基础工作机制是消息发送者可以将消息发送到 Kafka 上指定的 Topic，而消息消费者，可以从指定的 Topic 上消费消息。

> *(此处原文档含有一张架构示意图)*

首先，可以使用 Kafka 提供的客户端脚本创建 Topic：

```bash
# 创建 Topic
bin/kafka-topics.sh --create --topic test --bootstrap-server localhost:9092

# 查看 Topic
bin/kafka-topics.sh --describe --topic test --bootstrap-server localhost:9092
```

然后，启动一个消息发送者端，往一个名为 test 的 Topic 发送消息：

```bash
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
```

当命令行出现 `>` 符号后，随意输入一些字符。Ctrl+C 退出命令行。这样就完成了往 Kafka 发消息的操作。

如果不提前创建 Topic，那么在第一次往一个之前不存在的 Topic 发送消息时，消息也能正常发送，只是会抛出 `LEADER_NOT_AVAILABLE` 警告：

```
[oper@worker1 kafka_2.13-3.2.0]$ bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
>123
[2021-03-05 14:00:23,347] WARN [Producer clientId=console-producer] Error while fetching metadata with correlation id 1 : {test=LEADER_NOT_AVAILABLE} (org.apache.kafka.clients.NetworkClient)
[2021-03-05 14:00:23,479] WARN [Producer clientId=console-producer] Error while fetching metadata with correlation id 3 : {test=LEADER_NOT_AVAILABLE} (org.apache.kafka.clients.NetworkClient)
[2021-03-05 14:00:23,589] WARN [Producer clientId=console-producer] Error while fetching metadata with correlation id 4 : {test=LEADER_NOT_AVAILABLE} (org.apache.kafka.clients.NetworkClient)
>123
```

这是因为 Broker 端在创建完主题后，会通知 Clients 端 `LEADER_NOT_AVAILABLE` 异常。Clients 端接收到异常后，就会主动去更新元数据，获取新创建的主题信息。

然后启动一个消息消费端，从名为 test 的 Topic 上接收消息：

```bash
[oper@worker1 kafka_2.13-3.2.0]$ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test
qwe
qwe
123
123
123
^CProcessed a total of 5 messages
```

这样就完成了一个基础的交互。这其中，生产者和消费者并不需要同时启动。它们之间可以进行数据交互，但是又并不依赖于对方。没有生产者，消费者依然可以正常工作，反过来，没有消费者，生产者也依然可以正常工作。这也体现出了生产者和消费者之间的解耦。

如果想要查看这个脚本的详细参数，可以直接访问这个脚本，不配置任何参数即可。

### 4、其他消费模式

> 注：原文档此处编号跳至 4。

之前我们通过 Kafka 提供的生产者和消费者脚本，启动了一个简单的消息生产者以及消息消费者。实际上，Kafka 还提供了丰富的消息消费方式。

#### 指定消费进度

通过 `kafka-console-consumer.sh` 启动的控制台消费者，会将获取到的内容在命令行中输出。如果想要消费之前发送的消息，可以通过添加 `--from-beginning` 参数指定：

```bash
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --from-beginning --topic test
```

如果需要更精确的消费消息，甚至可以指定从哪一条消息开始消费：

```bash
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --partition 0 --offset 4 --topic test
```

这表示从第 0 号 Partition 上的第四个消息开始读起。Partition 和 Offset 是什么呢，后面会介绍到。

### 3、理解消费者组

对于每个消费者，可以指定一个消费者组。Kafka 中的同一条消息，只能被同一个消费者组下的某一个消费者消费。而不属于同一个消费者组的其他消费者，也可以消费到这一条消息。在 `kafka-console-consumer.sh` 脚本中，可以通过 `--consumer-property group.id=testGroup` 来指定所属的消费者组。

例如，可以启动三个消费者组，来验证一下分组消费机制：

```bash
# 两个消费者实例属于同一个消费者组
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --consumer-property group.id=testGroup --topic test
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --consumer-property group.id=testGroup --topic test

# 这个消费者实例属于不同的消费者组
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --consumer-property group.id=testGroup2 --topic test
```

#### 查看消费者组的偏移量

接下来，还可以使用 `kafka-consumer-groups.sh` 观测消费者组的情况，包括它们的消费进度：

```bash
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group testGroup
```

从这里可以看到，Kafka 是以消费者组为单位来分别记录每个 Partition 上的消息偏移量的。而增加新的消费者组，并不会影响 Kafka 的消息数据，只是需要新增一条偏移量记录就可以了。所以，Kafka 的消息复读效率是很高的。

### 4、理解 Kafka 的消息传递机制

从之前的实验可以看到，Kafka 的消息发送者和消息消费者通过 Topic 这样一个逻辑概念来进行业务沟通。但是实际上，所有的消息是存在服务端的 Partition 这样一个数据结构当中的。

> *(此处原文档含有一张 Kafka 架构示意图)*

在 Kafka 的技术体系中，有以下一些概念需要先熟悉起来：

- **客户端 Client**：包括消息生产者 和 消息消费者。之前简单接触过。
- **消费者组**：每个消费者可以指定一个所属的消费者组，相同消费者组的消费者共同构成一个逻辑消费者组。每一个消息会被多个感兴趣的消费者组消费，但是在每一个消费者组内部，一个消息只会被消费一次。
- **服务端 Broker**：一个 Kafka 服务器就是一个 Broker。
- **话题 Topic**：这是一个逻辑概念，一个 Topic 被认为是业务含义相同的一组消息。客户端都通过绑定 Topic 来生产或者消费自己感兴趣的话题。
- **分区 Partition**：Topic 只是一个逻辑概念，而 Partition 就是实际存储消息的组件。每个 Partition 就是一个 Queue 队列结构。所有消息以 FIFO 先进先出的顺序保存在这些 Partition 分区中。

---

## 三、理解 Kafka 的集群工作机制

对于 Kafka 这样一个追求消息吞吐量的产品来说，集群基本上是必备的。接下来，我们就动手搭建一个 Kafka 集群，并来理解一下 Kafka 集群的工作机制。

### 1、搭建 Kafka 集群

Kafka 的集群架构大体是这样的：

> *(此处原文档含有一张集群架构示意图)*

#### 为什么要用集群？

单机服务下，Kafka 已经具备了非常高的性能。TPS 能够达到百万级别。但是，在实际工作中使用时，单机搭建的 Kafka 会有很大的局限性。

**一方面：消息太多，需要分开保存。** Kafka 是面向海量消息设计的，一个 Topic 下的消息会非常多，单机服务很难存得下来。这些消息就需要分成不同的 Partition，分布到多个不同的 Broker 上。这样每个 Broker 就只需要保存一部分数据。这些分区的个数就称为分区数。

**另一方面：服务不稳定，数据容易丢失。** 单机服务下，如果服务崩溃，数据就丢失了。为了保证数据安全，就需要给每个 Partition 配置一个或多个备份，保证数据不丢失。Kafka 的集群模式下，每个 Partition 都有一个或多个备份。Kafka 会通过一个统一的 Zookeeper 集群作为选举中心，给每个 Partition 选举出一个主节点 Leader，其他节点就是从节点 Follower。主节点负责响应客户端的具体业务请求，并保存消息。而从节点则负责同步主节点的数据。当主节点发生故障时，Kafka 会选举出一个从节点成为新的主节点。

**最后**：Kafka 集群中的这些 Broker 信息，包括 Partition 的选举信息，都会保存在额外部署的 Zookeeper 集群当中，这样，Kafka 集群就不会因为某一些 Broker 服务崩溃而中断。Kafka 也提供了另外一种不需要 Zookeeper 的集群机制，Kraft 集群。这种方式会在后面进行介绍。

#### 准备实验环境

准备三台同样的 CentOS 服务器，预先安装好了 JDK，并关闭防火墙：

```bash
service firewalld stop
systemctl disable firewalld
```

分别配置机器名 worker1、worker2、worker3：

```bash
vi /etc/hosts
# worker1: 192.168.221.81
# worker2: 192.168.221.92
# worker3: 192.168.221.03
# (IP 地址换成自己的)
```

接下来我们就动手部署一个 Kafka 集群，来体验一下 Kafka 是如何面向海量数据进行横向扩展的。我们先来部署一个基于 Zookeeper 的 Kafka 集群。其中，选举中心部分，Zookeeper 是一种多数同意的选举机制，允许集群中少数节点出现故障。因此，在搭建集群时，通常都是采用 3、5、7 这样的奇数节点，这样可以最大化集群的高可用特性。在后续的实验过程中，我们会在三台服务器上都部署 Zookeeper 和 Kafka。

#### 1、部署 Zookeeper 集群

这里采用之前单独下载的 Zookeeper 来部署集群。Zookeeper 是一种多数同意的选举机制，允许集群中少半数节点出现故障。因此，在搭建集群时，通常采用奇数节点，这样可以最大化集群的高可用特性。在后续的实现过程中，我们会在三台服务器上都部署 Zookeeper。

先将下载下来的 Zookeeper 解压到 `/app/zookeeper` 目录。然后进入 conf 目录，修改配置文件。在 conf 目录中，提供了一个 `zoo_sample.cfg` 文件，这是一个示例文件。我们只需要将这个文件复制一份 `zoo.cfg`（`cp zoo_sample.cfg zoo.cfg`），修改下其中的关键配置就可以了。其中比较关键的修改参数如下：

```properties
# Zookeeper 的本地数据目录，默认是 /tmp/zookeeper。这是 Linux 的临时目录，随时会被删掉。
dataDir=/app/zookeeper/data

# Zookeeper 的服务端口
clientPort=2181

# 集群节点配置
server.1=192.168.221.81:2888:3888
server.2=192.168.221.92:2888:3888
server.3=192.168.221.103:2888:3888
```

其中，`clientPort=2181` 是对客户端开放的服务端口。集群配置部分，`server.x` 这个 x 就是节点在集群中的 myid。后面的 2888 端口是集群内部数据传输使用的端口。3888 是集群内部进行选举使用的端口。

接下来将整个 Zookeeper 的应用目录分发到另外两台机器上。然后需要构建对应的 myid 文件：

```bash
# 进入配置的 data 目录
cd /app/zookeeper/data
# 生成 myid 文件
echo 1 > myid
```

这个 myid 文件的内容就是在 `zoo.cfg` 中配置的对应的 `server.id`。

接下来可以在三台机器上都启动 Zookeeper 服务了：

```bash
bin/zkServer.sh --config conf start
```

启动完成后，使用 jps 指令可以看到一个 QuorumPeerMain 进程就表示服务启动成功。三台机器都启动完成后，可以查看下集群状态：

```
[root@hadoop02 zookeeper-3.5.8]# bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /app/zookeeper/zookeeper-3.5.8/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost.
Mode: leader
```

这其中 Mode 为 leader 就是主节点，follower 就是从节点。

#### 2、部署 Kafka 集群

Kafka 服务并不需要进行选举，因此也没有奇数台服务的建议。部署 Kafka 的方式跟部署 Zookeeper 差不多，就是解压、配置、启服务三板斧。首先将 Kafka 解压到 `/app/kafka` 目录下。

然后进入 config 目录，修改 `server.properties`。这个配置文件里面的配置项非常多，下面列出几个要重点关注的配置：

```properties
# broker 的全局唯一编号，不能重复，只能是数字。
broker.id=0

# 服务监听地址
listeners=PLAINTEXT://worker1:9092

# 数据文件地址。同样默认是给的 /tmp 目录。
log.dirs=/app/kafka/logs

# 默认的每个 Topic 的分区数
num.partitions=1

# zookeeper 的服务地址
zookeeper.connect=worker1:2181,worker2:2181,worker3:2181

# 可以选择指定 zookeeper 上的基础节点。
# zookeeper.connect=worker1:2181,worker2:2181,worker3:2181/kafka
```

`broker.id` 需要每个服务器上不一样，分发到其他服务器上时，要注意修改一下。多个 Kafka 服务注册到同一个 Zookeeper 集群上的节点，会自动组成集群。

配置文件中的注释非常细致，可以关注一下。下面是 `server.properties` 文件中比较重要的核心配置：

| Property | Default | Description |
|----------|---------|-------------|
| broker.id | 0 | broker 的"名字"，你可以选择任意你喜欢的数字作为 id，只要 id 是唯一的即可。每个 broker 都可以用一个唯一的非负整数 id 进行标识。 |
| log.dirs | /tmp/kafka-logs | Kafka 存放数据的路径。这个路径并不是唯一的，可以是多个，路径之间只需要使用逗号分隔即可；每当创建新 partition 时，都会选择在包含最少 partitions 的路径下进行。 |
| listeners | PLAINTEXT://127.0.0.1:9092 | server 接受客户端连接的端口，ip 配置 Kafka 本机 ip 即可。 |
| zookeeper.connect | localhost:2181 | Zookeeper 连接地址。hostname:port。如果是 Zookeeper 集群，用逗号连接。 |
| log.retention.hours | 168 | 每个日志文件删除之前保存的时间。 |
| num.partitions | 1 | 创建 Topic 的默认分区数。 |
| default.replication.factor | 1 | 自动创建 Topic 的默认副本数量。 |
| min.insync.replicas | 1 | 当 producer 设置 acks 为 -1 时，min.insync.replicas 指定 replicas 的最小数目（必须确认每一个 replica 的写数据都是成功的），如果这个数目没有达到，producer 发送消息会产生异常。 |
| delete.topic.enable | false | 是否允许删除主题。 |

接下来就可以启动 Kafka 服务了。启动服务时需要指定配置文件：

```bash
bin/kafka-server-start.sh -daemon config/server.properties
```

`-daemon` 表示后台启动 Kafka 服务，这样就不会占用当前命令窗口。通过 jps 指令可以查看 Kafka 的进程。

### 2、理解服务端的 Topic、Partition 和 Broker

接下来可以对比一下之前的单机服务，快速理解 Kafka 的集群当中核心的 Topic、Partition、Broker。

```bash
# 创建一个分布式的 Topic
[oper@worker1 bin]$ ./kafka-topics.sh --bootstrap-server worker1:9092 --create \
  --topic disTopic --partitions 4 --replication-factor 2
Created topic disTopic.

# 列出所有的 Topic
[oper@worker1 bin]$ ./kafka-topics.sh --bootstrap-server worker1:9092 --list
consumer_offsets
disTopic

# 查看 Topic 详情
[oper@worker1 bin]$ ./kafka-topics.sh --bootstrap-server worker1:9092 --describe --topic disTopic
Topic: disTopic	TopicId: X6pDZgTy10tQ	PartitionCount: 4	ReplicationFactor: 2	Configs: segment.bytes=1073741824
	Topic: disTopic	Partition: 0	Leader: 2	Replicas: 2,1	Isr: 2,1
	Topic: disTopic	Partition: 1	Leader: 1	Replicas: 1,0	Isr: 1,0
	Topic: disTopic	Partition: 2	Leader: 0	Replicas: 0,2	Isr: 0,2
	Topic: disTopic	Partition: 3	Leader: 2	Replicas: 2,0	Isr: 2,0
```

从这里可以看到：

1. **--create 创建集群**：可以指定一些补充的参数。大部分的参数都可以在配置文件中指定默认值。
   - `partitions` 参数表示分区数，这个 Topic 下的消息会分别存入这些不同的分区中。示例中创建的 disTopic 指定了四个分区，也就是说这个 Topic 下的消息会划分为四个部分。
   - `replication-factor` 表示每个分区有几个备份。示例中创建的 disTopic 指定了每个 partition 有两个备份。

2. **--describe 查看 Topic 信息**：
   - `Partition` 参数列出了四个 partition，后面带有分区编号，用来标识这些分区。
   - `Leader` 表示这一组 partition 中的 Leader 节点是哪一个。这个 Leader 节点就是负责响应客户端请求的主节点。从这里可以看到，Kafka 中的每一个 Partition 都会分配 Leader，也就是说每个 Partition 都有不同的节点来负责响应客户端的请求，这样就可以将客户端的请求做到尽量的分散。
   - `Replicas` 参数表示这个 partition 的多个备份是分配在哪些 Broker 上的，也称为 AR。这里的 0、1、2 就对应配置集群时指定的 broker.id。但是，Replicas 列出的只是一个逻辑上的分配情况，并不关心数据实际是不是按照这个分配。甚至有些节点服务挂了之后，Replicas 中也依然会列出节点的 ID。
   - `ISR` 参数表示 partition 的实际分配情况。它是 AR 的一个子集，只列出那些当前还存活、能够正常同步数据的那些 Broker 节点。

接下来，我们还可以查看 Topic 下的 Partition 分布情况。在 Broker 上，与消息联系最为紧密的，其实就是 Partition 了。之前在配置 Kafka 集群时指定了一个 `log.dirs` 属性，指向了一个服务器上的日志目录。进入这个目录，就能看到每个 Broker 的实际数据承载情况。

> *(此处原文档含有一张日志目录截图)*

从这里可以看到，Broker 上的一个 Partition 对应了日志目录中的一个目录。而这个 Partition 上的所有消息，就保存在这个对应的目录当中。

从整个过程可以看到，Kafka 当中，Topic 是一个数据集合的逻辑单元。同一个 Topic 下的数据，实际上是存储在 Partition 分区中的，Partition 就是数据存储的物理单元。而 Broker 是 Partition 的物理载体，这些 Partition 分区会尽量均匀地分配到不同的 Broker 机器上。而之前接触到的 offset，就是每个消息在 partition 上的偏移量。

> *(此处原文档含有一张 Topic/Partition/Broker 关系示意图)*

#### 这样设计解决了什么问题？

1. Kafka 设计需要支持海量的数据，而这样庞大的数据量，一个 Broker 是存不下的。那就拆分成多个 Partition，每个 Broker 只存一部分数据。这样极大地扩展了集群的吞吐量。

2. 每个 Partition 保留了一部分的消息副本，如果放到一个 Broker 上，就容易出现单点故障。所以就给每个 Partition 设计 Follower 节点，进行数据备份，从而保证数据安全。另外，多备份的 Partition 设计也提高了读取消息时的并发度。

3. 在同一个 Topic 的多个 Partition 中，会产生一个 Partition 作为 Leader。这个 Leader Partition 会负责响应客户端的请求，并将数据往其他 Partition 分发。

---

## 四、章节总结：Kafka 集群的消息流转模型

经过上面的实验，我们接触到了很多 Kafka 中的概念。将这些基础概念整合起来，就形成了 Kafka 集群的整体结构。这次我们先把这个整体结构梳理清楚，后续再一点点去了解其中的细节。

> *(此处原文档含有一张 Kafka 集群整体结构示意图)*

1. **Topic** 是一个逻辑概念，Producer 和 Consumer 通过 Topic 进行业务沟通。

2. Topic 并不存储数据，Topic 下的数据分为多组 **Partition**，尽量平均地分散到各个 Broker 上。每组 Partition 包含 Topic 下一部分的消息。每组 Partition 包含一个 **Leader Partition** 以及若干个 **Follower Partition** 进行备份，每组 Partition 的个数称为备份因子（replication factor）。

3. Producer 将消息发送到对应的 Partition 上，然后 Consumer 通过 Partition 上的 **Offset** 偏移量，记录自己所属消费者组 Group 在当前 Partition 上消费消息的进度。

4. Producer 发送给一个 Topic 的消息，会由 Kafka 推送给所有订阅了这个 Topic 的消费者组进行处理。但是在每个消费者组内部，只会有一个消费者实例处理这一条消息。

5. 最后，Kafka 的 Broker 通过 Zookeeper 组成集群。然后在这些 Broker 中，需要选举产生一个担任 **Controller** 角色的 Broker。这个 Controller 的主要任务就是负责 Topic 的分配以及后续管理工作。在我们实验的集群中，这个 Controller 实际上是通过 ZooKeeper 产生的。