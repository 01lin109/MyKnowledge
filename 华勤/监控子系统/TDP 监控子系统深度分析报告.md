
> 基于 `tdp-server/monitor/` 包的完整代码分析

---

## 监控子系统是干什么的：

- 7×24 小时自动持续采集 不用人工值守，持续抓取指标、日志、调用链路、中间件状态（ESB、MQ、数据库等），Arthas 做不到长期持续采集。
- 自动阈值检测 + 主动告警收敛 识别异常（大量 NPE、接口超时、消息堆积、CPU 爆满），自动推送钉钉 / 邮件；支持告警合并、限流抑制，避免告警风暴，这是单纯日志工具没有的能力。
- 持久化存储历史数据 长期留存异常记录、性能曲线、报错堆栈，支持回溯复盘，事后查几小时 / 几天前的故障现场。
- 全局可视化观测 统一大盘查看全集群、多服务整体状态，快速判断故障影响范围，而不是单台机器单点查看。
- 辅助故障定位 告警附带上下文：异常服务、接口、堆栈、机器 IP、时间，大幅缩短排查时间；发现告警后，再用 Arthas 上机器深挖根因。

## _一、这个子系统解决什么问题_

### 1.1 监控子系统常见问题：

#### 一、告警问题

1. 告警风暴：同类异常高频重复推送，淹没重要故障
    
2. 误报/漏报：阈值不合理、采集断点，要么乱报要么不报
    
3. 无分级无人认领：告警混杂，责任不清晰
    

#### 二、采集层问题

1. 数据丢失断档，故障无留存无法复盘
    
2. 采集过量消耗业务服务器性能
    
3. 日志/指标格式不统一，多系统数据割裂
    

#### 三、存储性能问题

1. 海量日志指标存储成本高、磁盘易打满
    
2. 大盘、历史日志查询卡顿
    
3. 监控组件无高可用，自身宕机全线失明
    

#### 四、定位能力不足

1. 指标孤立，无链路联动，无法快速定位根因
    
2. 告警缺少堆栈、机器、接口等上下文信息
    
3. 历史数据留存短，故障难以回溯
    

#### 五、运维集成问题

1. 多监控平台割裂，切换繁琐
    
2. 权限管控弱，规则易被随意改动
    
3. 和Arthas等线上诊断工具无法联动
    

### 1.2 核心痛点

一个企业级 HR 系统在生产环境中面临三类"看不见"的问题：

|痛点|具体表现|后果|
|---|---|---|
|**连接池耗尽**|HikariCP 连接池在高峰期被短暂打满，但运维只看瞬时值时已经恢复正常|间歇性请求超时，难以复现和定位|
|**慢 SQL 积累**|单次慢查询不超过告警阈值，但累积效应导致响应变慢|用户感知卡顿，但监控报表无异常|
|**错误风暴**|同一个 NPE 在高并发下每秒触发数千次，每次都发告警邮件/推送|告警淹没真正的问题，运维疲劳|

### 1.3 设计目标

监控子系统需要同时解决三个问题：

1. **采集什么** — 系统/JVM 指标、数据库连接池指标、慢 SQL、异常日志
    
2. **怎么采集** — 不能影响业务性能，不能阻塞请求线程
    
3. **怎么上报** — 去重压缩，避免告警风暴，通过 ESB 推送到 HMP 监控平台
    

---

## 二、整体架构

### 2.1 分层架构

```
┌─────────────────────────────────────────────────────┐
│                    调度层 (Scheduler)                  │
│           MonitorScheduler — @Scheduled 触发          │
└──────────────┬──────────────────────┬────────────────┘
               │                      │
       ┌───────▼────────┐    ┌───────▼────────┐
       │  指标采集层      │    │  数据推送层      │
       │  Collectors     │    │  Adapters       │
       └───────┬────────┘    └───────┬────────┘
               │                      │
       ┌───────▼──────────────────────▼────────┐
       │            核心服务层 (Service)          │
       │     MonitorReportService — 编排/去重     │
       └───────┬───────────────────────────────┘
               │
       ┌───────▼───────────────────────────────┐
       │            拦截层 (Interceptor)          │
       │  SqlMonitorInterceptor — MyBatis 拦截    │
       │  GlobalExceptionHandler — 异常捕获       │
       └───────────────────────────────────────┘
               │
       ┌───────▼───────────────────────────────┐
       │            配置层 (Config)              │
       │  MonitorAutoConfiguration — 全局配置     │
       │  MonitorProperties — 属性绑定           │
       └───────────────────────────────────────┘
```

### 2.2 三条数据流

子系统内部有三条独立的数据流，各自解决不同的监控问题：

|数据流|触发方式|采集频率|数据内容|推送目标|
|---|---|---|---|---|
|**系统/JVM 指标**|`@Scheduled` 定时|60 秒|CPU、内存、堆、GC、线程|HMP — `METRIC`|
|**数据库连接池**|`@Scheduled` 定时|60 秒|活跃/空闲连接、等待线程、DB 端连接数|HMP — `DATASOURCE`|
|**错误日志**|异常触发（被动）|实时/去重后|慢 SQL、SQL 异常、业务异常|HMP — `ERROR_LOG`|

---

## 三、完整流转链路详解（系统/JVM）

### 3.1 系统/JVM 指标流

```
MonitorScheduler.collectMetric()          // @Scheduled，延迟120秒后首次执行
    │
    ├── SystemMetricsCollector.collect()  // 读取 OS MXBean
    │   ├── CPU 使用率 (com.sun.management.OperatingSystemMXBean)
    │   ├── 内存使用率 (MemoryMXBean)
    │   └── 启动时长 (RuntimeMXBean)
    │
    ├── JvmMetricsCollector.collect()     // 读取 JVM MXBean
    │   ├── 堆内存: used/max/percent
    │   ├── 非堆内存
    │   ├── 线程数
    │   └── GC 次数增量 (AtomicLong.getAndSet)
    │
    ├── MonitorReportService.collectAndPushSnapshot()
    │   ├── 合并两个 Collector 的结果为 MonitorSnapshotDto
    │   ├── 设置 systemCode、instanceName、hostName、collectTime
    │   └── pushToHmp(METRIC, dto)
    │
    └── TdpHmpDataPushAdapter.push()
        ├── 构建 EsbReqHeader (serviceName = REST_HMP_MONITOR_METRIC)
        ├── JSON.parseArray() 还原为 Java 对象 (防双序列化)
        └── 提交到有界队列 (1000容量，DiscardOldestPolicy)
            └── 守护线程 → EsbReqClient → ESB Gateway → HMP 平台
```

**关键设计点：**

- **增量 GC 计数**：`AtomicLong.getAndSet()` 原子读取并重置，两次采集之间的差值才是有意义的 GC 活动量
- **GC 名称启发式分类**：通过子串匹配（"young"/"new"/"eden" → Young GC，"old"/"tenured" → Old GC）适配 G1/ZGC/CMS/Parallel 等不同收集器

> 我的问题：采集的定时任务——@Scheduled注解——监控范围 ：单机 OR 分布式

**解答：**

监控子系统是单机的，但是通过架构设计实现了多实例汇聚

- 单机层面

@Scheduled 确实只在当前 JVM 进程内执行。每个实例独立采集自己的 CPU、内存、GC、连接池指标。TdpHmpDataPushAdapter 也是每个实例各自推送。

- 汇聚

（1）instanceName 唯一标识每个实例

MonitorAutoConfiguration 的 @PostConstruct 自动生成实例名：

POD_NAME (K8s) → HOSTNAME → WebServer IP:port → 本地 IP:port

每条推送数据都携带 systemCode + instanceName，HMP 平台收到后按实例名聚合。3 个实例就是 3 条独立的指标记录。

（2）推送到同一个HMP平台

// TdpHmpDataPushAdapter — 每个实例都推到同一个 ESB 地址

esbReqClient.sendRequest(esbRequest); // → ESB Gateway → HMP

HMP 平台收到所有实例的数据后，按 instanceName 维度做聚合展示。所以你在 HMP 上能看到的是一个仪表盘展示 N 个实例的指标，而不是只有某一台的。

### 3.2当前方案局限性：

#### 局限一：多实例重复采集

`MonitorScheduler` 使用 `@Scheduled` 定时触发，每个 JVM 实例独立执行全量采集和推送。代码中没有任何分布式锁、Leader 选举或实例间协调机制。

- 3 个实例 → 3 倍采集开销 + 3 倍 ESB 推送流量
- 唯一的"控制"是 `isEnabled()` 开关，但这是模块级开关，不是实例级调度

#### 局限二：无采集限流/采样

`TdpHmpDataPushAdapter` 使用单线程 + 1000 容量有界队列 + `DiscardOldestPolicy`。队列满时最旧数据直接丢弃，不是排队等待或采样降级。

- 指标数据（METRIC、DATASOURCE）完全没有去重或采样
- 错误日志有时间窗口去重（`tryDeduplicateAndSchedule`），但仅限 ERROR_LOG 类型
- 高并发场景下 ESB 推送会被丢弃，HMP 端看不到这些数据

#### 局限三：采集逻辑耦合业务进程

采集逻辑和业务代码运行在同一个 JVM 进程。`SqlMonitorInterceptor` 拦截业务 SQL，采集发生在业务请求线程中。

缓解措施存在但有限：

- `DatasourceMetricsCollector` 采样线程设为 `NORM_PRIORITY - 1`（低优先级）
- `TdpHmpDataPushAdapter` 用独立线程池，不阻塞业务线程
- 但 JVM 指标采集在 Spring `@Scheduled` 线程池中执行，与业务共享

#### 局限四：无离线缓存、无重试

`TdpHmpDataPushAdapter.doPush()` 推送失败时只记日志，没有本地文件缓存、没有重试机制、没有失败队列留存。`DiscardOldestPolicy` 意味着队列满时数据直接被丢弃，没有第二次机会。

#### 局限五：采集生命周期绑定业务服务

整个监控模块通过 `@ConditionalOnProperty` 控制生命周期。业务服务挂了 → `@Scheduled` 也停了 → HMP 收不到数据。没有任何外部探活或 Sidecar 机制来弥补。

### 3.3设计取舍的合理性分析

#### 考量一：_业务 SLA 优先级远高于监控 SLA_

代码中反复出现的注释暴露了核心约束：

```

// TdpHmpDataPushAdapter.java:83
// ⚠️ 核心：立即提交到异步线程池并返回，不阻塞调用方（HTTP 请求线程）

// GlobalExceptionHandler.java:90
// 异步上报 HMP（绝不阻塞当前请求线程）

// SqlMonitorInterceptor.java:100-104
// 监控推送失败绝不向上抛异常！只记日志，保证主业务不受影响
```

这是刻意的架构决策：**监控挂了可以容忍，业务挂了不行**。用 `DiscardOldestPolicy` 丢弃数据而不是阻塞，用 try-catch 吞掉异常而不是抛出，都是服务于这个最高原则。

#### 考量二：运维环境的混合部署约束

从 `MonitorAutoConfiguration.generateInstanceName()` 的实现可以看出部署环境：

```
// 优先级：POD_NAME → HOSTNAME → WebServer IP → 本地 IP
// 还要过滤 Docker/K8s 虚拟网卡（docker*, veth*, br-*, cni*, cali*, flannel*）
```

部署环境是混合的——既有 K8s 容器，也有传统物理机/虚拟机。在这种环境下，要实现分布式 Leader 选举（Redis 锁、ZooKeeper），需要引入额外的中间件依赖。对于一个"尽力而为"的监控模块来说，这个依赖成本不值得。

#### 考量三：责任边界清晰

HMP 数据走的是企业级 ESB Hub，不是直连。这意味着：

- ESB 本身有流量控制和路由能力
- HMP 平台大概率有多实例部署，接收端有扩展能力
- 推送失败的监控数据，ESB 网关可能有重试机制

TDP 团队只管"推给 ESB"，ESB 后面的事由平台团队负责。这是**责任边界清晰**的设计。

#### 考量四：成本与收益的工程权衡

|设计选择|收益|代价|
|---|---|---|
|单机采集|零外部依赖，一个 JAR 搞定|多实例重复|
|异步丢弃|业务零感知|数据丢失|
|无重试|简单可靠|故障时段断档|
|内嵌进程|部署简单，无 Sidecar|绑定业务生命周期|

对于 HR 系统来说，监控的核心诉求是**"出事了能看到是哪台机器、什么异常"**，而不是"精确到秒级的全链路可观测"。上述取舍在该场景下是合理的。

#### 适用规模边界

这些局限在什么规模下会成为真正的问题：

|实例规模|当前方案状态|建议|
|---|---|---|
|1-5 个实例|完全够用，ESB 能承受|保持现状|
|10-20 个实例|开始有压力|评估 ESB 吞吐，考虑错峰采集|
|50+ 实例|需要架构升级|考虑 Sidecar 或集中式采集 Agent|

对于华勤这种规模的企业 HR 系统，实例数大概率在个位数，当前设计是够用的。这些"问题"更多是理论上的风险，而非实际痛点。

### 3.4最佳范式：

#### 一、标准 Agent 监控极简整体方案

1. 业务服务：仅暴露 Actuator 指标接口，无定时上报代码
2. 每台宿主机 / Pod 部署独立 Agent 进程
3. Agent 定时拉取本机所有业务实例指标，本地聚合、压缩、缓存
4. Agent 统一批量推送至 ESB / 监控平台 HMP，携带 instance 区分多实例数据
5. 平台按实例维度存储、展示、告警

#### 二、对应原有 4 个问题及 Agent 解决手段

1. 多实例同步推送产生 ESB 流量洪峰、重复执行采集代码 Agent 统一拉取，错开采集时间，批量打包上报，减少请求量，消除并发脉冲。
2. 采集占用业务 JVM 资源，业务故障导致监控断档 Agent 独立进程，资源隔离；业务卡死、GC 阻塞不影响采集。
3. 网络 / ESB 波动直接丢指标，无重试缓存 Agent 自带磁盘缓存、失败自动重试，指标不易丢失。
4. 修改采集规则必须发版重启业务 Agent 配置独立，调整采集频率、指标过滤无需改动业务代码、不用发版。

## 四、完整流转链路详解（数据库连接池）

### 4.1 数据库连接池指标流

```
MonitorScheduler.collectDatasource()      // @Scheduled，延迟120秒后首次执行
    │
    └── DatasourceMetricsCollector.collectAll()
        │
        ├── 策略1: JMX MBean 采集
        │   ├── 查找 com.zaxxer.hikari:type=Pool (*)
        │   ├── 通过 HikariPoolMXBean 获取: activeConnections, idleConnections, threadsAwaiting
        │   └── 反查 Spring DataSource 获取: maxPoolSize, minIdle
        │
        ├── 策略2: Spring Bean 降级 (JMX 不可用时)
        │   ├── 获取所有 HikariDataSource Bean
        │   └── 通过反射读取 HikariPoolMXBean
        │
        ├── applyWindowAggregation()      // 滑动窗口峰值替换
        │   ├── 替换 activeCount/idleCount/waitThread 为窗口内最大值
        │   └── 重新计算 connectionUsage
        │
        ├── queryDbSideMetrics()          // 直连数据库查询
        │   ├── SHOW VARIABLES LIKE 'max_connections'
        │   └── SHOW STATUS LIKE 'Threads_connected'
        │
        └── MonitorReportService → pushToHmp(DATASOURCE) → 同上推送链路
```

### **4.2滑动窗口算法（核心）**

```java
// SlidingWindowSampler — 环形缓冲区
int[] activeSamples = new int[60];  // 60个采样点 × 10秒 = 10分钟窗口
int writeIndex = 0;                  // 写指针，到60后回绕
int sampleCount = 0;                 // 已采集数（≤60）

void record(int active, int idle, int waiting) {
    activeSamples[writeIndex % 60] = active;  // O(1) 写入
    writeIndex++;
}

WindowStats getWindowStats() {
    int maxActive = 0;
    for (int i = 0; i < Math.min(sampleCount, 60); i++) {
        maxActive = Math.max(maxActive, activeSamples[i]);  // O(n) 扫描
    }
    // ... 计算 maxIdle, maxWaiting, 平均值
}
```

**为什么需要滑动窗口：**

连接池的峰值可能只持续几百毫秒。如果采集间隔是 60 秒，很可能刚好错过峰值，监控报表显示"一切正常"。滑动窗口用 10 秒采样一次、保留 10 分钟历史，确保能捕获到短暂的连接池耗尽事件。

### 💡 那为什么不用“上报时现查”？

你可能会想：“反正上报时也要取最大值，为什么不等到上报那一刻再一次性查数据库？”

这恰恰是滑动窗口的另一个巧妙之处：采集和上报是解耦的。

- 采集是轻量级的内存操作，对业务影响极小。
- 如果等到上报时才去查，那相当于把60次查询压缩到一次，但那次查询会非常重，而且如果数据库刚好在那一刻抖动，你可能什么都查不到。

用内存缓存的方式，数据是持续、平稳地采集进来的，上报只是把已经准备好的数据取走，既快又稳。

## 五、完整流转链路详解（错误日志流、慢SQL）

### 5.1错误日志流

```
业务代码抛出异常
    │
    ├── 路径A: SQL 异常
    │   │
    │   ├── SqlMonitorInterceptor.intercept()
    │   │   ├── 检查 isSqlMonitoringActive() (sqlEnabled && hmpEnabled)
    │   │   ├── 执行 SQL，捕获 Throwable
    │   │   ├── sanitizeSql() — O(n) 状态机脱敏
    │   │   └── SqlMonitorCollector.recordSqlError()
    │   │       ├── unwrapException() — 最多遍历10层异常链
    │   │       ├── appendCallerLocation() — 跳过 JDK/Spring/MyBatis 帧
    │   │       └── MonitorReportService.pushErrorLog(dto)
    │   │
    │   └── GlobalExceptionHandler 也会捕获该异常
    │       ├── isSqlRelatedException() — 遍历异常链检查类名
    │       ├── 如果是 SQL 相关 → 跳过 HMP 上报 (避免重复)
    │       └── 如果不是 SQL 相关 → 正常上报
    │
    └── 路径B: 非 SQL 异常 (业务异常、NPE 等)
        │
        └── GlobalExceptionHandler
            ├── 遍历异常链 (最多20层)，逐层检查类名
            ├── 映射错误码 (ERR_NPE, ERR_OOM, ERR_DEADLOCK...)
            ├── 映射用户友好消息 ("系统处理异常，请稍后重试")
            ├── 提取堆栈定位 (跳过框架帧，找业务代码位置)
            └── MonitorReportService.pushErrorLog(dto)
                    │
                    ├── 去重模式 (errorDedupEnabled=true)
                    │   ├── 构建去重键: "errorClass|errorMethod"
                    │   ├── ConcurrentHashMap.putIfAbsent() — CAS 创建窗口
                    │   ├── ErrorWindow.recordRepeat() — occurCount++
                    │   └── flushScheduler — 30秒后触发 flushWindow()
                    │       └── pushToHmp(ERROR_LOG)
                    │
                    └── 直推模式 (errorDedupEnabled=false)
                        └── pushToHmp(ERROR_LOG) — 直接推送
```

**SQL 脱敏状态机（O(n) 无回溯）**

```
正常字符 ──'──→ 字符串字面量状态
  │                │
  │──'──→ '' 转义（保持在字符串状态）
  │──其他──→ 替换为 ?
  │
  ├─── '──'──→ 单行注释状态 ──\n──→ 正常
  │
  └─── '/'──'*'──→ 块注释状态 ──'*'/'──→ 正常
```

为什么不用正则？代码注释记录了一次线上事故：2026-06-01，复杂 SQL 在正则脱敏时触发 catastrophic backtracking，导致全站 StackOverflowError。手写状态机保证 O(n) 时间复杂度，且无任何回溯。

### 5.2慢SQL流

```
SqlMonitorInterceptor.intercept()
    │
    ├── 执行 SQL
    ├── 计算 executeTime
    │
    ├── 如果 executeTime > 1000ms (可配置)
    │   └── SqlMonitorCollector.recordSlowSql()
    │       ├── 构建 MonitorErrorLogDto
    │       │   ├── errorLevel = "WARN"
    │       │   ├── errorCode = "WARN_SLOW_SQL"
    │       │   ├── errorSummary = "[慢SQL 2350ms] UserMapper.selectById(QUERY)"
    │       │   └── errorClass = "SlowSqlMonitor"
    │       └── MonitorReportService.pushErrorLog(dto) → 同去重/直推链路
    │
    └── 无异常且未超阈值 → 不记录
```

---

## 六、核心设计原理

### 6.1 三层条件激活

|层级|配置项|控制范围|粒度|
|---|---|---|---|
|**一级：模块级**|`hmp.monitor.enabled=true`|整个 `MonitorAutoConfiguration` 类|20+ Bean 全部创建/全部不创建|
|**二级：功能级**|`hmp.monitor.sql-enabled=true`|SQL 拦截器内部逻辑|Bean 仍创建，但不采集|
|**三级：特性级**|`hmp.monitor.task-enabled=true`|任务监控 Bean|`@ConditionalOnProperty` + `@ConditionalOnBean` 双重守卫|

**关闭时的零开销保证：** 当 `enabled=false` 时，整个 `MonitorAutoConfiguration` 类不会被 Spring 加载。没有任何 Bean 被创建，没有任何 `@Scheduled` 方法被注册，没有任何 `@Intercepts` 被激活。不是"创建后跳过逻辑"，而是"根本不创建"。

### 6.2 监控永不阻塞业务

这是贯穿整个子系统的第一设计原则：

|组件|阻断机制|
|---|---|
|`SqlMonitorInterceptor`|`isSqlMonitoringActive()` 快速路径短路返回|
|`MonitorReportService`|所有推送包在 try-catch 中，异常被吞掉|
|`TdpHmpDataPushAdapter`|有界队列(1000) + `DiscardOldestPolicy`，队列满时丢弃最旧数据|
|`GlobalExceptionHandler`|异步守护线程上报，不阻塞 HTTP 响应|
|`DatasourceMetricsCollector`|采样线程设为 `NORM_PRIORITY - 1`，低于业务线程|
|所有 Collector|每个指标采集独立 try-catch，单个失败不影响其他指标|

### 6.3 告警去重算法

```
错误发生 → 提取去重键 ("errorClass|errorMethod")
    │
    ├── 首次出现
    │   ├── ConcurrentHashMap.putIfAbsent(key, new ErrorWindow())
    │   │   └── 如果并发创建，CAS 只保留一个
    │   ├── window.recordRepeat()  // occurCount = 1
    │   └── flushScheduler.schedule(flushWindow, 30秒)
    │
    └── 重复出现 (30秒内)
        ├── window.recordRepeat()  // occurCount++
        └── 不创建新任务 (复用同一个 flushWindow)
            │
            └── 30秒后
                ├── flushWindow()
                │   ├── synchronized(window) + volatile flushed 防重入
                │   ├── pushToHmp(ERROR_LOG) — 一条记录，occurCount = N
                │   └── pendingErrors.remove(key)
                │
                └── 结果: N次相同错误 → 1条上报 (携带 occurCount=N)
```

**缓存溢出降级：** 当 `pendingErrors.size() > maxPending(200)` 时，新错误绕过去重，直接实时推送。防止在极端错误场景下（如数据库不可用导致大量异常）内存无限增长。

### 6.4 异常链智能处理

`GlobalExceptionHandler` 的异常处理包含三个智能判断：

**1) SQL 异常去重判断：**

```
异常链遍历 (最多20层)
    → 检查每一层的 className 是否匹配 SQL/JDBC/MyBatis 模式
    → 如果匹配：说明 SqlMonitorInterceptor 已经上报过了，跳过
    → 如果不匹配：说明是业务异常，正常上报
```

这避免了同一个 SQL 异常在拦截器层和全局异常处理器层被重复上报。

**2) 堆栈智能定位：**

```
遍历 stackTrace
    → 跳过: sun.*, java.lang.*, org.apache.ibatis.*, org.springframework.*
    → 跳过: $Proxy*, CGLIB$$*, .monitor.*
    → 返回第一个匹配 errorLogBasePackage 的帧
```

输出如 `UserService.getUserById(UserService.java:142)`，而不是 `DefaultSqlSession.selectList(DefaultSqlSession.java:135)`。

**3) 错误码自动推断：**

```
NPE  → ERR_NPE
OOM  → ERR_OOM
死锁 → ERR_DEADLOCK
连接拒绝 → ERR_CONN_REFUSED
...
```

结构化的错误码便于 HMP 平台做聚合分析和告警规则配置。

---

## 七、与其他方案的对比

### 7.1 与 Spring Boot Actuator 的对比

|维度|TDP Monitor|Spring Boot Actuator|
|---|---|---|
|**采集方式**|自定义 Collector，直接读 MXBean|通过 Micrometer 框架适配|
|**推送目标**|企业内部 HMP 平台（通过 ESB）|Prometheus/Grafana/Datadog 等|
|**连接池监控**|双策略(JMX+Bean) + 滑动窗口 + DB 端查询|仅 Micrometer HikariCP binder|
|**SQL 监控**|MyBatis Interceptor + O(n) 脱敏 + 慢 SQL 检测|需额外集成 P6Spy 或自定义|
|**错误去重**|内置时间窗口去重，N→1 聚合|无，依赖告警工具侧去重|
|**配置方式**|Apollo 热配置，单开关激活|application.properties|
|**侵入性**|零侵入（AOP + Interceptor）|需要添加依赖和配置端点|

**TDP 方案的优势：** 面向企业内部监控平台定制，包含连接池滑动窗口、SQL 脱敏、错误去重等 Actuator 不直接提供的能力。

**Actuator 的优势：** 生态更丰富，社区支持更好，与 Prometheus/Grafana 集成更成熟。

### 7.2 与 APM 工具（SkyWalking/Pinpoint）的对比

| 维度        | TDP Monitor                         | SkyWalking / Pinpoint |
| --------- | ----------------------------------- | --------------------- |
| **定位**    | 轻量级指标采集 + 错误上报                      | 全链路分布式追踪              |
| **链路追踪**  | 无（仅单机指标）                            | 完整的分布式 Trace/Span     |
| **字节码增强** | 不使用                                 | Agent 基于字节码增强         |
| **性能开销**  | 极低（仅读 MXBean + MyBatis Interceptor） | 较高（类加载、方法拦截）          |
| **部署方式**  | 应用内嵌，无需额外 Agent                     | 需要独立 Agent 或 Sidecar  |
| **数据存储**  | 推送到外部 HMP 平台                        | 自有存储（ES/B+Tree）       |
| **适用场景**  | 单机健康监控、错误聚合                         | 微服务调用链分析              |

**TDP 方案的定位：** 不是全链路 APM，而是专注于"单机健康"和"错误聚合"的轻量监控。适合已有 ESB 基础设施的企业环境。

### 7.3 与 ELK/EFK 日志方案的对比

|维度|TDP Monitor|ELK/EFK|
|---|---|---|
|**数据类型**|结构化指标 + 聚合错误|原始日志全文|
|**存储成本**|低（仅推送聚合后的数据）|高（全文索引，存储量大）|
|**查询能力**|依赖 HMP 平台的查询能力|Kibana 全文搜索|
|**实时性**|秒级推送|秒级采集，分钟级可搜索|
|**错误去重**|内置|需要 Kibana 聚合或 Logstash 去重|
|**慢 SQL**|内置检测 + 脱敏|需要解析慢查询日志|

**TDP 方案的优势：** 不存储原始日志，只推送聚合后的结构化数据，存储成本低。内置的错误去重避免了 ELK 中常见的"日志风暴"问题。

### 7.4 方案选型建议

```
需要什么？
    │
    ├── 单机健康监控 + 错误聚合 + 已有 ESB 基础设施
    │   └── ✅ TDP Monitor 方案
    │
    ├── 微服务调用链追踪 + 分布式系统诊断
    │   └── ✅ SkyWalking / Pinpoint
    │
    ├── 云原生 + Prometheus 生态
    │   └── ✅ Spring Boot Actuator + Micrometer
    │
    ├── 全文日志搜索 + 审计分析
    │   └── ✅ ELK / EFK
    │
    └── 全面覆盖
        └── ✅ TDP Monitor (指标) + SkyWalking (链路) + ELK (日志)
```

---

## 八、可复用的设计模式总结

### 8.1 可直接迁移的模式

|模式|代码位置|复用场景|
|---|---|---|
|**滑动窗口环形缓冲**|`DatasourceMetricsCollector.SlidingWindowSampler`|任何需要"峰值检测"的指标采集（连接池、线程池、队列长度）|
|**O(n) SQL 脱敏状态机**|`SqlMonitorInterceptor.sanitizeSql()`|任何需要清理 SQL 中的字符串字面量/注释的场景|
|**错误时间窗口去重**|`MonitorReportService.ErrorWindow`|任何高频重复错误的告警聚合（日志、APM、邮件告警）|
|**单开关模块激活**|`MonitorAutoConfiguration` 的 `@ConditionalOnProperty`|任何可选的横切模块（限流、熔断、审计日志）|
|**异步有界队列推送**|`TdpHmpDataPushAdapter`|任何需要"fire-and-forget"异步上报的场景|
|**异常链智能定位**|`GlobalExceptionHandler.extractCallerLocation()`|任何需要从堆栈中自动定位业务代码位置的场景|

### 8.2 架构设计原则

1. **监控永不阻塞业务** — 每个采集/推送路径都有独立的异常处理和异步降级
2. **单开关零开销** — 关闭时不只是"跳过逻辑"，而是"不创建 Bean"
3. **生产事故驱动** — SQL 脱敏状态机就是线上 StackOverflow 事故的直接产物
4. **显式 Bean 声明** — 不依赖 `@ComponentScan`，完全控制 Bean 创建顺序
5. **SPI 扩展点** — `TaskDataSource` 接口 + `@ConditionalOnMissingBean` 适配器，支持业务系统按需接入

