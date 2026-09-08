# SQL异常/慢SQL 采集完整链路分析

# 一、采集入口 SqlMonitorInterceptor（MyBatis 插件层）

```
业务代码调用 Mapper 方法
│
└─ MyBatis Executor（执行器）
        │  被 SqlMonitorInterceptor 代理拦截
        │  @Intercepts 拦截两个签名：
        │    ① Executor.update(MappedStatement, Object)          → INSERT/UPDATE/DELETE
        │    ② Executor.query(MappedStatement, Object, RowBounds, ResultHandler) → SELECT
        │
        ├─ 快速路径：isSqlMonitoringActive() == false → 直接 proceed()，零开销
        ├─ 排除表检查：isExcludedTable(ms.getId()) 命中 → 直接放行
        │
        └─ 正式路径（try/finally 包裹，异常原样上抛，绝不吞掉）
             ├─ t0 = System.currentTimeMillis()
             ├─ invocation.proceed()          ← 真正执行 SQL
             ├─ 捕获 Throwable 存入 sqlException 后 re-throw（不影响原异常流转）
             └─ finally:
                  ├─ executeTime = now - t0
                  ├─ 异常分支  → recordSqlError(...)   （异常 SQL，ERROR 级）
                  ├─ 慢SQL分支 → executeTime >= 1000ms → recordSlowSql(...)（WARN 级）
                  └─ 采集动作本身再包一层 try-catch，失败只 warn，不影响业务
```

## 1.1 为什么挂在 MyBatis 层而不是 JDBC/HTTP 层

| 挂载点 | 优点 | 缺点 |
|---|---|---|
| JDBC DataSource 代理 | 最底层，能拿到真实 SQL | 需要包装数据源，多数据源侵入大 |
| **MyBatis Interceptor（选用）** | 拿得到 MappedStatement id、SQL类型、参数对象，与框架天然集成 | 只覆盖 MyBatis 执行的 SQL |
| HTTP 层 GlobalExceptionHandler | 已有通道 | 拿不到 SQL 语句本身，且 SQL 异常会先被业务层捕获/包装，堆栈失真 |

与 HTTP 报告的联动：`GlobalExceptionHandler.handleSystemException()` 中有一道 `isSqlRelatedException()` 过滤——遍历异常链 20 层，类名命中 sql/jdbc/ibatis/Persistence 关键字就**跳过 HTTP 层上报**。原因：SQL 异常已由 `SqlMonitorInterceptor` 在 MyBatis 层第一时间上报过，HTTP 层再报一次就是重复告警。

## 1.2 三层开关（为什么是三层）

```
hmp.monitor.enabled      → MonitorProperties 总开关（监控模块所有 Bean 的创建前提）
hmp.monitor.sql-enabled  → SQL 监控分开关（默认 true）
两者同时为 true          → isSqlMonitoringActive() 返回 true
```

关键点：`MonitorAutoConfiguration` 标注 `@ConditionalOnProperty(hmp.monitor.enabled=true)`，
开关关闭时拦截器 Bean 根本不存在；而运行期快速路径只做两个布尔判断就放行，
对正常 SQL 的开销是两次 boolean 读取 + 一次方法调用。

# 二、装配与注册链路（比较特殊的两段式）

```
① MonitorAutoConfiguration（@ConditionalOnProperty enabled=true）
     ├─ @Bean SqlMonitorCollector(reportService, properties)
     └─ @Bean SqlMonitorInterceptor(collector, properties)

② SqlMonitorConfig（独立配置类，避免与 MybatisPlusConfiguration 循环依赖）
     └─ ApplicationRunner sqlMonitorInterceptorRegistrar()
          ├─ 触发时机：Spring Boot 完全启动后（此时 SqlSessionFactory 已就绪）
          ├─ @Autowired(required=false) 注入拦截器（enabled=false 时为 null，直接跳过）
          ├─ 遍历容器中所有 SqlSessionFactory（支持多数据源）
          ├─ 去重检查：interceptor == 已注册实例 则跳过
          └─ factory.getConfiguration().addInterceptor(interceptor)
```

## 2.1 为什么用 ApplicationRunner 而不是 @Bean 直接塞进去

- MyBatis-Plus 的 `SqlSessionFactory` 由 starter 自动装配创建，直接在其创建过程中插入拦截器会产生 Bean 创建顺序/循环依赖问题。
- ApplicationRunner 在**应用完全启动后**执行，此时所有 SqlSessionFactory 都已初始化，注册动作是纯内存操作，无时序风险。
- 单独拆一个 `SqlMonitorConfig` 配置类，把"监控模块的 Bean 定义"和"注册动作"解耦，注释中明确说明是为了避免与 MybatisPlusConfiguration 循环依赖。

# 三、慢SQL 与 异常SQL 的判定

| 维度 | 慢SQL | 异常SQL |
|---|---|---|
| 判定条件 | `executeTime >= slowSqlThreshold`（默认 1000ms，Apollo 可配） | `proceed()` 抛出任意 Throwable |
| 优先级 | 异常优先：同一次执行两者都满足时**只报异常**（if/else if） | — |
| errorLevel | WARN | ERROR |
| errorCode | WARN_SLOW_SQL | ERR_SQL_EXCEPTION |
| errorClass | 固定 "SlowSqlMonitor" | 解包后的真实 DB 异常类简名（如 SQLSyntaxErrorException） |
| errorSummary | `[慢SQL 2350ms] UserMapper.selectById(SELECT)` | `[SQL异常-SELECT] UserMapper.insert · Column 'xxx' cannot be null`（原因超120字截断） |
| errorStack | 无 | 根因类名+message + 前3帧堆栈 |

计时注意：用的是 `try/catch + finally` 而不是 AOP 环绕——异常路径也保证计时和上报，同时**把原异常原样 re-throw**，监控层对业务完全透明。

# 四、SQL 获取与脱敏（本链路最有含金量的部分）

## 4.1 获取 SQL

```
getSql(ms, parameter)
  → ms.getBoundSql(parameter).getSql()   ← 拿到带 ? 占位符的预编译 SQL
  → sanitizeSql(sql)                      ← 脱敏
  失败兜底：返回 "UNKNOWN_SQL"（获取失败不能阻断上报）
```

注意：这里拿的是 `BoundSql.getSql()`（占位符版本），**不拼接真实参数值**，本身就是第一层脱敏——用户手机号、密码等经 `#{}` 传入的参数天然不会出现在上报内容里。脱敏主要针对 XML/注解 SQL 中**硬编码的字面量**（如 `${tableName}` 注入后的值、写死的条件值、注释里的业务信息）。

## 4.2 脱敏状态机 sanitizeSql —— O(n) 零回溯

处理三类内容：

```
① 单行注释  -- xxx        → 整段删除（跳到行尾）
② 多行注释  /* xxx */     → 整段删除（跳到 */）
③ 字符串字面量 'abc'      → 整体替换为单个 ?（正确处理 '' 转义）
④ 普通字符               → 原样保留
最后：replaceAll("\\s+", " ") 折叠空白 + trim
```

⚠️ **为什么不用正则（真实事故，面试重点）**

```
事故：2026-06-01 全站请求 StackOverflowError
根因：旧实现用 replaceAll + 复杂正则（类似 (?:[^']|'')*' 的模式）匹配字符串字面量。
     Java 正则引擎是 NFA 回溯型，对某些长 SQL 输入产生灾难性回溯
     （catastrophic backtracking），递归深度打爆线程栈。
方案：改为手写单遍状态机，逐字符 while 循环 + 显式状态跳转，
     时间复杂度严格 O(n)，没有任何回溯路径。
```

细节处理：
- `''`（SQL 中转义的单引号）→ 判定为"仍在字符串内部"，继续消费，不会提前截断。
- 脱敏过程整体再包 try-catch：脱敏自身异常时**降级返回原文**（宁可多报不漏报），只 warn 不中断上报。
- 结尾的 `replaceAll("\\s+", " ")` 虽然也是正则，但 `\s+` 无嵌套量词、无交替分支，线性匹配，无回溯风险。

# 五、收集器 SqlMonitorCollector —— 把原始信息组装成上报 DTO

## 5.1 异常解包 unwrapException（拿到真实 DB 错误）

MyBatis/Spring 会把底层 SQLException 层层包装：

```
MyBatisSystemException(PersistenceException(BadSqlGrammarException(SQLSyntaxErrorException)))
                                        ↓ unwrap 后
                              SQLSyntaxErrorException   ← 真正有信息量的异常
```

```
解包规则（最多穿透 10 层，防循环引用 current.getCause() != current）：
  cause 类名命中 WRAPPER_EXCEPTIONS（InvocationTargetException /
  PersistenceException / MyBatisSystemException）→ 继续向内
  当前异常有自己非空且与 cause 不同的 message → 停止（它本身就是有价值的异常）
  解不出来 → 返回原异常兜底
```

## 5.2 业务调用位置定位 appendCallerLocation

```
Thread.currentThread().getStackTrace() 遍历当前线程堆栈：
  跳过 JDK 内部（sun. / java.lang. / jdk.internal.）
  跳过反射代理（$Proxy / CGLIB$$）
  跳过 MyBatis（org.apache.ibatis. / org.mybatis.）
  跳过 Spring AOP（org.springframework.aop. / cglib.）
  跳过监控模块自己（.monitor.）
  第一个以 errorLogBasePackage 开头的帧 = 业务调用方
  → 写入 errorLine（行号），配合 errorMethod 展示定位到行
```

价值：报出的是"**哪行业务代码触发了这条 SQL**"，而不是 MyBatis 内部帧——排查时直接跳到代码位置。

The user wants a brief explanation of the SQL exception DTO assembly logic (SqlMonitorCollector). Let me recall what I know about SqlMonitorCollector from earlier exploration. I referenced it earlier:

- [SqlMonitorCollector.recordSqlError() L102] and [recordSlowSql() L65]
- unwrapException L232-277
- shortMapperId L289-299

I should read the actual file to be accurate since I need to give precise details. The user asked for a brief explanation, but I must be accurate. Let me read SqlMonitorCollector.java.

Now I can give a brief explanation of the SQL exception DTO assembly. The user wants it brief (简要说明). Let me structure it:

1. Entry: SqlMonitorInterceptor → recordSlowSql / recordSqlError (both on the business request thread, synchronously in the finally block)
2. Two methods assemble the DTO differently (WARN vs ERROR)
3. Key helper methods: unwrapException, shortMapperId, findCallerFromThread/appendCallerLocation, buildShortStack, getErrorMessage
4. Then pushErrorLog → dedup etc. (same shared path)

Note differences vs HTTP path:
- Uses new MonitorErrorLogDto() directly (not builder) — same DTO class though
- errorClass for slow SQL is fake "SlowSqlMonitor" (not a real exception class)
- errorMethod = Mapper short ID, not business frame
- errorLine = business caller line from thread stack (appendCallerLocation, with basePkg filter from config)
- errorSummary format differs
- errorStack = root cause + first 3 frames (only for SQL exception; slow SQL doesn't set it)
- clientIp = "-" placeholder
- Time fields: recordSqlError sets them explicitly; recordSlowSql doesn't (let pushErrorLog fill defaults)
- Runs synchronously on business thread (inside finally of the interceptor), whole thing wrapped in try-catch

Keep it brief with a comparison table maybe. Use file links. Brief!
SQL 路径的 DTO 组装在 [SqlMonitorCollector.java](file:///d:/hq_work_code/tdp-server/tdp-server/tdp-server-webapp/src/main/java/com/huaqin/tdp/server/monitor/interceptor/SqlMonitorCollector.java) 中，分两个入口（注意：它跑在**业务请求线程**上，是拦截器 finally 块里的同步调用，整个方法体 try-catch 包裹，失败只记日志）：

## 5.3DTO组装
### 两个入口，两种 DTO

**1. 慢SQL [recordSlowSql() L57-84](file:///d:/hq_work_code/tdp-server/tdp-server/tdp-server/tdp-server-webapp/src/main/java/com/huaqin/tdp/server/monitor/interceptor/SqlMonitorCollector.java#L57-L84)**（无异常对象，纯超时事件）

| 字段 | 值 |
|---|---|
| errorLevel / errorCode | `"WARN"` / `"WARN_SLOW_SQL"` |
| errorClass | **硬编码 `"SlowSqlMonitor"`**——不是真实异常类，是造出来的标识 |
| errorMethod | Mapper 短 ID：`UserMapper.selectById` |
| errorSummary | `[慢SQL 1520ms] UserMapper.selectById(SELECT)` |
| clientIp | `"-"` 占位（没有 request 对象） |
| errorStack | **不设置**（慢SQL没有异常堆栈） |

**2. SQL异常 [recordSqlError() L95-140](file:///d:/hq_work_code/tdp-server/tdp-server/tdp-server/tdp-server-webapp/src/main/java/com/huaqin/tdp/server/monitor/interceptor/SqlMonitorCollector.java#L95-L140)**

| 字段 | 值 |
|---|---|
| errorLevel / errorCode | `"ERROR"` / `"ERR_SQL_EXCEPTION"` |
| errorClass | 解包后根因的**简名**，如 `SQLSyntaxErrorException` |
| errorMethod | Mapper 短 ID（同上） |
| errorSummary | `[SQL异常-UPDATE] OrderMapper.updateStatus · Data truncated...`（根因消息截 120 字符） |
| errorStack | [buildShortStack()](file:///d:/hq_work_code/tdp-server/tdp-server/tdp-server/tdp-server-webapp/src/main/java/com/huaqin/tdp/server/monitor/interceptor/SqlMonitorCollector.java#L273-L287)：`根因类名: 消息` + **前 3 帧**（精简堆栈，对比 HTTP 路径整个不放堆栈） |
| 三个时间字段 + occurCount | 自己显式设 `now()` / 1（慢SQL路径不设，留给 pushErrorLog 补默认值） |

### 三个关键辅助方法

- **[unwrapException() L228-258](file:///d:/hq_work_code/tdp-server/tdp-server/tdp-server/tdp-server-webapp/src/main/java/com/huaqin/tdp/server/monitor/interceptor/SqlMonitorCollector.java#L228-L258)**：解包拿真实 DB 错误。策略和 HTTP 路径的 findRootCause 不同——不是无脑走到链尾，而是沿 WRAPPER_EXCEPTIONS 名单（`InvocationTargetException`/`PersistenceException`/`MyBatisSystemException`）逐层剥壳，**一旦当前异常有自己的非空消息就提前停**（最多 10 层）。因为 MyBatis 链尾可能是无消息的底层异常，中间层反而信息量大
- **[shortMapperId() L288-301](file:///d:/hq_work_code/tdp-server/tdp-server/tdp-server/tdp-server-webapp/src/main/java/com/huaqin/tdp/server/monitor/interceptor/SqlMonitorCollector.java#L288-L301)**：`com.x.UserMapper.selectById` → `UserMapper.selectById`，只留最后两段
- **[findCallerFromThread() L174-214](file:///d:/hq_work_code/tdp-server/tdp-server/tdp-server/tdp-server-webapp/src/main/java/com/huaqin/tdp/server/monitor/interceptor/SqlMonitorCollector.java#L174-L214)**：从**当前线程堆栈**（不是异常堆栈）找业务调用帧，跳过 JDK/反射/MyBatis/Spring AOP/monitor 自身，且额外要求命中 `errorLogBasePackage` 配置的业务包名才返回——比 HTTP 路径的“第一个非框架帧”多了一层包名白名单，防误判更严。找到后行号写入 errorLine（**但类名方法名丢弃不用**，errorMethod 仍是 Mapper ID）

### 组装完之后

和 HTTP 路径汇合同一条链路：`pushErrorLog()` 补默认值 → 去重 key = `真实根因类名|Mapper短ID` → 窗口聚合 → ESB。SQL 文本（已脱敏）**不进 DTO 任何字段**，只在摘要里体现类型和耗时。

# 六、上报链路 MonitorReportService.pushErrorLog

```
pushErrorLog(errorLogDto)
  ├─ 填充默认值：systemCode / instanceName / level兜底FATAL / occurCount / 三个时间字段
  ├─ ★ 时间窗口聚合去重（errorDedupEnabled，默认开启）
  │     ├─ dedupeKey = errorClass + "|" + errorMethod（同方法同异常 = 同一问题）
  │     ├─ 命中已有窗口 → occurCount++，本次不推送（被吸收）
  │     ├─ 未命中 → ConcurrentHashMap.putIfAbsent 创建窗口（并发安全，
  │     │           竞争失败方累加到赢家窗口）
  │     │           + 调度 30s 后 flushWindow 刷出
  │     ├─ 缓存超限（>200 个窗口）→ 降级为实时推送（不丢数据，只放弃聚合）
  │     └─ flushWindow：synchronized + flushed 标志防重复刷出，
  │            复用首次完整 DTO，仅覆盖 occurCount/firstOccurTime/lastOccurTime
  └─ toJsonArray(dto)  ← HMP 服务端要求入参必须是 JSON 数组，单条也包 []
       → pushToHmp(ERROR_LOG, json)
            ├─ 策略1（优先）：HmpDataPushAdapter Bean → TdpHmpDataPushAdapter
            ├─ 策略2（降级）：反射调用 esbClientBeanName(esbReqClient).pushToEsb(...)
            └─ 策略3（兜底）：打印缺策略警告，返回 false（不抛异常）
```

## 6.1 为什么要有时间窗口聚合（N→1）

SQL 异常的典型形态是**惊群**：一条索引失效的慢SQL，30 秒内可能被打中几百次，每次都推 ESB 就是飞书轰炸。

方案取舍：
- key 用 `errorClass|errorMethod`——粒度是"同一方法里的同一种异常"，既不会把不同问题合并，也不会漏掉重复问题。
- 窗口 30s（默认）：首次发生延迟最多 30s 通知，换取 N 次合并为 1 次（occurCount 带着真实次数）。
- 缓存上限 200 做内存兜底，超限**降级为实时推送而非丢弃**——告警噪音可以接受，数据不能丢。

## 6.2 TdpHmpDataPushAdapter —— 异步化最后一公里

```
pushToHmp(serviceName, jsonData)
  → executor.submit(() -> doPush(...))   立即返回，不阻塞业务线程

线程池设计：
  单线程 + 有界队列(1000) + 守护线程 + DiscardOldestPolicy（队列满丢最旧，防 OOM）
  → 监控推送是"尽力而为"，不需要并发度，要的是绝不影响主业务

doPush 内两个易错点（注释里明确标了 ⚠️）：
  ① jsonData 是已序列化的 JSON 字符串，直接 setReqData(String) 会被
     Jackson 二次序列化成 "[{...}]" 字符串而非数组 → 必须先 JSON.parseArray
     解回 List 再放入请求体
  ② EsbReqHeader 所有字段必须填满（requestId/serviceName/version/
     sourceSystem/targetSystem），任何 null 触发 ESB 网关 002545

响应校验：esbCode == "000000" 才算成功
异常处理：doPush 内部全量 try-catch，推送失败只记 error 日志，绝不向上抛
```

# 七、关键配置项汇总

| 配置 | 默认值 | 作用 |
|---|---|---|
| `hmp.monitor.enabled` | false | 总开关，false 时整个监控模块 Bean 不创建 |
| `hmp.monitor.sql-enabled` | true | SQL 监控分开关 |
| `hmp.monitor.slow-sql-threshold` | 1000ms | 慢SQL 判定阈值 |
| `hmp.monitor.excluded-sql-tables` | 空 | 排除表列表（MappedStatement id contains 匹配） |
| `hmp.monitor.error-log-base-package` | 无 | 业务代码基础包，用于堆栈定位 |
| `hmp.monitor.error-dedup-enabled` | true | 时间窗口聚合开关 |
| `hmp.monitor.error-dedup-window-seconds` | 30 | 聚合窗口大小 |
| `hmp.monitor.error-dedup-max-pending` | 200 | 窗口缓存上限（内存兜底） |

# 八、发现的问题 / 扩展点

1. **excludedTables 是启动时快照**：在构造器里一次性拷贝成 HashSet，Apollo 热更新 `excludedSqlTables` 后拦截器不会感知，需重启（或改造成每次实时读取 properties）。
2. **SqlMonitorCollector.slowSqlThreshold 是死字段**：`@Value` 注入了但从未使用（判定在拦截器里做了），可清理。
3. **慢SQL 的 errorClass 固定为 "SlowSqlMonitor"**：去重 key = `SlowSqlMonitor|UserMapper.selectById`，同一 mapper 的慢SQL 在窗口内正确聚合；但不同耗时档位的慢SQL 无法区分频次分布。
4. **脱敏只处理了字符串字面量**：数字字面量（如 `LIMIT 100`、硬编码 id）会原样上报，数字一般不敏感可接受；若 SQL 中出现 `${}` 拼接的敏感数值需另行处理。
5. **tdp-common 中的 MonitorSqlLogDto 未接入本链路**：该 DTO 定义了 sqlStatement/dsCode/executeTime 等完整 SQL 监控字段，但实际链路走的是 MonitorErrorLogDto（错误日志通道）。未来若 HMP 需要独立的 SQL 统计上报通道，此 DTO 是预留的扩展点。
6. **查询参数不在上报内容中**：`BoundSql.getSql()` 是占位符版本，`#{}` 参数天然脱敏；这是"设计上就不采集"而非"采集后清洗"，从源头规避了敏感数据外泄。
