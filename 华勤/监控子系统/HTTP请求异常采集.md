# 一、采集入口 GlobalExceptionHandler

```
HTTP 请求抛出异常
│
├─ ① HqBusinessException      → 业务异常，直接返回 e.getMessage()，【不上报HMP】
├─ ② HqBusinessCheckException → 校验异常，直接返回消息，【不上报 HMP】
├─ ③ MethodArgumentNotValidException → 参数校验，拼接字段错误，【不上报 HMP】
└─ ④ Exception（兜底 handleSystemException）
         ├─ 记 error 日志
         ├─ SQL/DB 相关异常 → 跳过（SqlMonitorInterceptor 已在 MyBatis 层报过）
         ├─ 其他异常 → 异步上报 HMP
         └─ 返回友好提示 Rest.failure(...) 给前端

     关键点：只有 ④ 兜底分支会上报监控，①②③ 是"预期内异常"，只返回前端、不告警。
     这是合理的——业务异常（如"该用户不存在"）不应触发飞书轰炸。
```

## 1.1业务异常（不上报/不上报）

_业务异常（HqBusinessException 等）→ 返回具体业务错误信息给前端_

- HqBusinessException ❌ 不上报 直接返回错误信息给前端
- HqBusinessCheckException ❌ 不上报 直接返回校验信息给前端
- MethodArgumentNotValid ❌ 不上报 拼接所有字段校验错误返回

兜底/其他异常 不上报HMP

- 其他 Exception ✅ 上报 异步推送监控 + 友好提示

## 1.2是否上报reason

#### 1.2.1HqBusinessException：业务规则违例

这是**"预期内的业务失败"**，属于正常业务逻辑的分支之一，而非系统缺陷。典型场景：

- 某区域OD未配置 无法正常审批 ---> 这是业务规则判断的正常结果
- 用户权限不足 无法操作 ---> 这是业务风控的正常拦截

监控价值判断：如果上报这类异常，会产生严重的告警噪音：

- 每天可能产生数万条"权限不足"、"密码错误"等告警
- 运维和开发人员会产生告警疲劳，真正的致命异常会被淹没
- 这类异常是给用户看的，不是给运维/开发看的

衡量标准：一个异常如果是产品需求文档中明确描述过的错误场景，就不该上报监控。

#### 1.2.2 HqBusinessCheckException：业务前置校验失败

这是**"业务参数校验失败"**，比 HqBusinessException 更轻量，通常是接口入参的业务合法性检查：

- 手机号格式不符合业务要求（虽然格式正确但不是国内号段）
- 验证码过期或错误
- 提交的时间范围不合法（开始时间晚于结束时间）
- 身份证号校验位不通过

与系统异常的本质区别：这类异常表示调用方没有按照业务规则传参，不是系统代码有bug。

类比：就像银行柜台工作人员告诉你"开户需要身份证原件"——这是业务规则提醒，不是银行系统崩溃了。

#### 1.2.3 MethodArgumentNotValidException

这是 Spring Validation 框架层的校验异常（`@Valid` / `@Validated` 注解触发），甚至还没进入业务代码层：

```
// 举例：Controller 层的入参声明
@PostMapping("/user")
public Rest<?> createUser(@Valid @RequestBody UserCreateRequest req) { ... }

// Request 中的校验注解
@NotBlank(message = "用户名不能为空")
@Length(max = 20, message = "用户名不超过20字")
private String username;

@Email(message = "邮箱格式不正确")
private String email;
```

当 `username` 为空或 `email` 格式错误时，Spring 在调用 Controller 方法之前就抛出此异常，业务代码一行都没执行。

不上报的核心理由：

1. 纯粹是调用方问题：前端/客户端传参不符合 schema 约定
2. 发生频率极高：用户输入场景下（注册表单、搜索条件）每分钟可能触发成百上千次
3. 对定位系统问题零价值：出现这种异常只能说明调用方没遵守接口文档，不能说明服务端有任何 bug

#### 1.2.3 兜底异常 上报 （一般需要上报的异常)

NullPointerException // 必报，代码 bug

SQLException / DataAccessException // 必报，DB 问题

OutOfMemoryError / StackOverflowError // 必报，JVM 级故障

ConnectException / SocketTimeoutException // 必报，依赖故障

RuntimeException(无业务语义的) // 必报，未知 bug

自定义系统级 SystemException // 必报，系统级故障

## 1.3不上报的业务异常共性&&常见无需上报异常

|公共特性|说明|
|---|---|
|可预期|客户端错误输入导致的，不是系统 bug，开发时就预见会发生|
|高频/批量|一旦触发往往是大量并发（爬虫、压测、用户批量误操作），上报会刷爆|
|业务语义清晰|异常类型本身已说明问题，不需要堆栈定位，日志里一眼能看懂|

**判断"异常是否该上报"的黄金五问**

总结一个通用决策框架，遇到任何异常类型都可以用这个框架判断：

|问题|**答"是" → 不上报**|**答"否" → 考虑上报**|
|---|---|---|
|**1. 这是代码 bug 吗？**|不是，是预期分支|是，代码逻辑有缺陷|
|**2. 开发/运维看到后需要行动吗？**|不需要，正常现象|需要，要查日志修问题|
|**3. 产品文档中是否描述过这个错误场景？**|描述过，是正常业务提示|没描述过，是意外情况|
|**4. 根因是服务端还是调用方/用户？**|调用方/用户输入问题|服务端处理逻辑问题|
|**5. 一天内可能出现超过 50 次吗？**|高频发生，是常态|低频，出现就是异常|

五个问题全部答"是" → 100% 不该上报，会产生告警噪音 五个问题全部答"否" → 必须上报，是严重系统异常 混合答案 → 根据权重判断，或只聚合统计不触发飞书告警

## 1.4 扩展点

MonitorProperties配置文件中留出了需要排除的异常类型可自由配置之后去进行实现

1. MonitorProperties 加 @RefreshScope ， MonitorAutoConfiguration 的 @Bean 方法去掉 static
2. ApolloConfigRefresh.onChange() 中加一行 refreshScope.refresh("monitorProperties")
3. GlobalExceptionHandler 注入 MonitorProperties ，在兜底方法中加一个 isExcludedException() 检查——遍历异常链，类名命中 excludeExceptions 列表就跳过上报
4. Apollo 控制台配置 hmp.monitor.exclude-exceptions[0]=xxx 即可随时增删排除项，不重启生效
5. 配置中心推送失败降级策略：

- Apollo 客户端本地缓存 — Apollo 推送失败或服务端不可用时，客户端使用本地缓存的上一份配置，不会丢配置。
- RefreshScope 的原子性 — refreshScope.refresh() 先创建新 Bean 实例，成功后才销毁旧的。如果重新绑定抛异常，旧实例保留，相当于自动回滚到上一个版本。
- 类名配错无害 — excludeExceptions 只是字符串列表做 contains() 匹配。写错了类名就是匹配不到，不会报错，等同于该项不生效，不影响其他项。
- 构造器兜底默认值 — MonitorProperties 构造器里预置了 3 个默认排除项。Apollo 上没配这个 key 时，字段保留构造器默认值；配了空列表则覆盖为空（等于全上报，是最宽松模式）。
- 唯一需要关注的极端场景 ：Apollo 配了语法错误的值导致 Spring 绑定异常（比如 List 格式写错）。这时 RefreshScope 创建新实例失败，旧实例保留—— 自动降级为上次成功的配置 ，不会让应用挂掉。
- 所以整体上： Apollo + RefreshScope + 构造器默认值 三层已经覆盖了回滚和降级，不需要额外代码。

# 二、兜底异常处理

## 2.1拦截+过滤

```
异常进入兜底 @ExceptionHandler(Exception.class)
  → 先打日志：[系统异常] 异常类名 + 请求URL
  → 过滤1：isSqlRelatedException() → 遍历异常链20层，类名命中 sql/jdbc/ibatis/Persistence 等关键字则跳过
  → 过滤2：（excludeExceptions 尚未启用）
  → 通过过滤 → 调 reportErrorAsync()
```

## 2.2异步化

```
reportErrorAsync()
  → new Thread("hmp-error-report") 直接创建线程（非线程池）
  → setDaemon(true) 守护线程，JVM 退出不阻塞
  → 内部 try-catch 全包裹，任何异常只 warn 不抛出
```

# 一、采集入口 GlobalExceptionHandler

```
HTTP 请求抛出异常
│
├─ ① HqBusinessException      → 业务异常，直接返回 e.getMessage()，【不上报HMP】
├─ ② HqBusinessCheckException → 校验异常，直接返回消息，【不上报 HMP】
├─ ③ MethodArgumentNotValidException → 参数校验，拼接字段错误，【不上报 HMP】
└─ ④ Exception（兜底 handleSystemException）
         ├─ 记 error 日志
         ├─ SQL/DB 相关异常 → 跳过（SqlMonitorInterceptor 已在 MyBatis 层报过）
         ├─ 其他异常 → 异步上报 HMP
         └─ 返回友好提示 Rest.failure(...) 给前端

     关键点：只有 ④ 兜底分支会上报监控，①②③ 是"预期内异常"，只返回前端、不告警。
     这是合理的——业务异常（如"该用户不存在"）不应触发飞书轰炸。
```

## 1.1业务异常（不上报/不上报）

_业务异常（HqBusinessException 等）→ 返回具体业务错误信息给前端_

- HqBusinessException ❌ 不上报 直接返回错误信息给前端
- HqBusinessCheckException ❌ 不上报 直接返回校验信息给前端
- MethodArgumentNotValid ❌ 不上报 拼接所有字段校验错误返回

兜底/其他异常 不上报HMP

- 其他 Exception ✅ 上报 异步推送监控 + 友好提示

## 1.2是否上报reason

#### 1.2.1HqBusinessException：业务规则违例

这是**"预期内的业务失败"**，属于正常业务逻辑的分支之一，而非系统缺陷。典型场景：

- 某区域OD未配置 无法正常审批 ---> 这是业务规则判断的正常结果
- 用户权限不足 无法操作 ---> 这是业务风控的正常拦截

监控价值判断：如果上报这类异常，会产生严重的告警噪音：

- 每天可能产生数万条"权限不足"、"密码错误"等告警
- 运维和开发人员会产生告警疲劳，真正的致命异常会被淹没
- 这类异常是给用户看的，不是给运维/开发看的

衡量标准：一个异常如果是产品需求文档中明确描述过的错误场景，就不该上报监控。

#### 1.2.2 HqBusinessCheckException：业务前置校验失败

这是**"业务参数校验失败"**，比 HqBusinessException 更轻量，通常是接口入参的业务合法性检查：

- 手机号格式不符合业务要求（虽然格式正确但不是国内号段）
- 验证码过期或错误
- 提交的时间范围不合法（开始时间晚于结束时间）
- 身份证号校验位不通过

与系统异常的本质区别：这类异常表示调用方没有按照业务规则传参，不是系统代码有bug。

类比：就像银行柜台工作人员告诉你"开户需要身份证原件"——这是业务规则提醒，不是银行系统崩溃了。

#### 1.2.3 MethodArgumentNotValidException

这是 Spring Validation 框架层的校验异常（`@Valid` / `@Validated` 注解触发），甚至还没进入业务代码层：

```
// 举例：Controller 层的入参声明
@PostMapping("/user")
public Rest<?> createUser(@Valid @RequestBody UserCreateRequest req) { ... }

// Request 中的校验注解
@NotBlank(message = "用户名不能为空")
@Length(max = 20, message = "用户名不超过20字")
private String username;

@Email(message = "邮箱格式不正确")
private String email;
```

当 `username` 为空或 `email` 格式错误时，Spring 在调用 Controller 方法之前就抛出此异常，业务代码一行都没执行。

不上报的核心理由：

1. 纯粹是调用方问题：前端/客户端传参不符合 schema 约定
2. 发生频率极高：用户输入场景下（注册表单、搜索条件）每分钟可能触发成百上千次
3. 对定位系统问题零价值：出现这种异常只能说明调用方没遵守接口文档，不能说明服务端有任何 bug

#### 1.2.3 兜底异常 上报 （一般需要上报的异常)

NullPointerException // 必报，代码 bug

SQLException / DataAccessException // 必报，DB 问题

OutOfMemoryError / StackOverflowError // 必报，JVM 级故障

ConnectException / SocketTimeoutException // 必报，依赖故障

RuntimeException(无业务语义的) // 必报，未知 bug

自定义系统级 SystemException // 必报，系统级故障

## 1.3不上报的业务异常共性&&常见无需上报异常

|公共特性|说明|
|---|---|
|可预期|客户端错误输入导致的，不是系统 bug，开发时就预见会发生|
|高频/批量|一旦触发往往是大量并发（爬虫、压测、用户批量误操作），上报会刷爆|
|业务语义清晰|异常类型本身已说明问题，不需要堆栈定位，日志里一眼能看懂|

**判断"异常是否该上报"的黄金五问**

总结一个通用决策框架，遇到任何异常类型都可以用这个框架判断：

|问题|**答"是" → 不上报**|**答"否" → 考虑上报**|
|---|---|---|
|**1. 这是代码 bug 吗？**|不是，是预期分支|是，代码逻辑有缺陷|
|**2. 开发/运维看到后需要行动吗？**|不需要，正常现象|需要，要查日志修问题|
|**3. 产品文档中是否描述过这个错误场景？**|描述过，是正常业务提示|没描述过，是意外情况|
|**4. 根因是服务端还是调用方/用户？**|调用方/用户输入问题|服务端处理逻辑问题|
|**5. 一天内可能出现超过 50 次吗？**|高频发生，是常态|低频，出现就是异常|

五个问题全部答"是" → 100% 不该上报，会产生告警噪音 五个问题全部答"否" → 必须上报，是严重系统异常 混合答案 → 根据权重判断，或只聚合统计不触发飞书告警

## 1.4 扩展点

MonitorProperties配置文件中留出了需要排除的异常类型可自由配置之后去进行实现

1. MonitorProperties 加 @RefreshScope ， MonitorAutoConfiguration 的 @Bean 方法去掉 static
2. ApolloConfigRefresh.onChange() 中加一行 refreshScope.refresh("monitorProperties")
3. GlobalExceptionHandler 注入 MonitorProperties ，在兜底方法中加一个 isExcludedException() 检查——遍历异常链，类名命中 excludeExceptions 列表就跳过上报
4. Apollo 控制台配置 hmp.monitor.exclude-exceptions[0]=xxx 即可随时增删排除项，不重启生效
5. 配置中心推送失败降级策略：

- Apollo 客户端本地缓存 — Apollo 推送失败或服务端不可用时，客户端使用本地缓存的上一份配置，不会丢配置。
- RefreshScope 的原子性 — refreshScope.refresh() 先创建新 Bean 实例，成功后才销毁旧的。如果重新绑定抛异常，旧实例保留，相当于自动回滚到上一个版本。
- 类名配错无害 — excludeExceptions 只是字符串列表做 contains() 匹配。写错了类名就是匹配不到，不会报错，等同于该项不生效，不影响其他项。
- 构造器兜底默认值 — MonitorProperties 构造器里预置了 3 个默认排除项。Apollo 上没配这个 key 时，字段保留构造器默认值；配了空列表则覆盖为空（等于全上报，是最宽松模式）。
- 唯一需要关注的极端场景 ：Apollo 配了语法错误的值导致 Spring 绑定异常（比如 List 格式写错）。这时 RefreshScope 创建新实例失败，旧实例保留—— 自动降级为上次成功的配置 ，不会让应用挂掉。
- 所以整体上： Apollo + RefreshScope + 构造器默认值 三层已经覆盖了回滚和降级，不需要额外代码。

# 二、兜底异常处理

## 阶段1：拦截+过滤

```
异常进入兜底 @ExceptionHandler(Exception.class)
  → 先打日志：[系统异常] 异常类名 + 请求URL ----> 给运维用
  → 过滤1：isSqlRelatedException() → 遍历异常链20层，类名命中 sql/jdbc/ibatis/Persistence 等关键字则跳过
  → 过滤2：（excludeExceptions 尚未启用）
  → 通过过滤 → 调 reportErrorAsync()
```

### 问题一：既然上报了为啥还要打印日志：

日志负责"完整、确定、本地的事实留痕"，HMP 上报负责"远端实时告警通知"——前者是后者的兜底 且信息更完整，删了日志等于把异常的最后一道安全网也撤了

异常链是啥？

```
Java 里抛异常时可以把"原始原因"挂在新异常上，形成一条单向链表：
catch (SQLException e) {
         throw new RuntimeException("查询失败", e);  // e 作为 cause 挂上去
  }

 调用 getCause() 就能拿到下一层。链表形如
     RuntimeException (顶层，GlobalExceptionHandler 捕获到的)
        └─ getCause() -> MyBatisSystemException   (Spring 翻译的)
             └─ getCause() -> SQLException        (JDBC 真正的)
                  └─ getCause() -> null
 为什么会有链：分层架构里每一层都习惯把下层异常"包装"成自己的类型往上抛（Controller 包 Service、Service 包
 Mapper、Spring 把 SQLException 翻译成 DataAccessException…），但都把原始异常保留为 cause，方便追溯根因。
```

### **问题二：为啥要设置成遍历15层**

这个数字是防御性上限 + 工程经验值，不是硬性规定：必须有上限（不能无限遍历）的原因：

1. 自引用：current != current.getCause()（line 121）挡住了 a.getCause() == a 这种自己指向自己的情况
2. 循环引用：但挡不住 a.getCause() == b && b.getCause() == a 这种互相引用的环（虽然 JDK 正常不会产生，但第三方库/序列化/反射搞出来的脏对象可能）。depth 计数器是最终兜底，保证循环时也能退出
3. 病态深链：重试框架、动态代理、多层反射偶尔会造出几十上百层的链，没上限会空耗 CPU

为什么是 15 这个值：

1. 真实深度：Spring + MyBatis + JDBC 的正常链路一般 3‑6 层；加上 Feign/ESB 重试、事务管理器、AOP 切面等额外包装，极端也就 10‑12 层
2. 15 也够用（真实链路极少超 12），但裕量偏小，换个框架版本就可能踩边界30 没意义：
3. 如果 20 层还没匹配到 SQL 关键字，说明这条链里根本就没有 SQL 异常，再多走 10 层只是白耗 CPU

所以 15、20、30 在 99.9% 的情况下行为完全一致，选 15 是“既安全裕量够、又不会无意义空转”的折中。 代码里同文件 findRootCause（line 218）用的是 maxDepth = 10--因为它只是找根因，链路不超 10 层就够了；而 isSqlRelatedException 是“扫描匹配”，要更保守一点留余量，所以取 20。两处数字不同，反映的是用途不同带来的裕量取舍，没有对错。

一句话：遍历链是为了穿透包装层找到真正的 SQL 异常；10 层是防止循环引用/病态深链的兜底上限，15~25 之间任何值实际效果都一样。

### **问题三:遍历异常链是同步的？**

为什么必须同步

1. 它是闸门，不是耗时操作：遍历异常链只是指针追逐(current = current.getCause())+类名字符串 contains匹配，无I/0、无网络，真实链路3~6层，微秒级就跑完。把它塞进异步反而要先派线程再判断，开销比直接同步算还大。
2. 异步线程的派发依赖它的结果：if(!isSqlRelatedException(ex)) 决定走 reportErrorAsync 还是 log.info跳过。如果判断本身异步，就得"派线程->判断->再决定要不要再派一个线程做上报"，逻辑拧巴且白白多一次线程切换。
3. 同步部分全程不阻塞：line87~106 这一整段同步代码里，没有任何网络/磁盘阻塞调用(log.error落盘是异步appender或极快的fsync，可忽略)。真正可能阻塞的只有pushErrorLog的HTTP推送--而它已经被推到守护线程里了。所以同步段对请求响应时间的影响几乎为零。

同步 vs 异步的边界在哪

整条链路里唯一异步的是 reportErrorAsync 派出的那个守护线程，它承载的是doReport→ pushErrorLog(line 209，HTTP推HMP/ESB，可能几十~几百毫秒甚至超时）。其余所有逻辑（日志、去重判断、根因查找、堆栈提取、DT0构建、友好提示构建）全是同步跑在请求线程上的轻量操作。

一句话：去重判断同步、快、必须先于上报完成；真正慢的HTTP上报才异步。这个分层是为了"不阻塞请求"和"决策准确"两个目标兼得。

## 阶段2：异步化

```
reportErrorAsync()
  → new Thread("hmp-error-report") 直接创建线程（非线程池）
  → setDaemon(true) 守护线程，JVM 退出不阻塞
  → 内部 try-catch 全包裹，任何异常只 warn 不抛出

  真实请求链路---两层异步
  [请求线程]
reportErrorAsync —new Thread("hmp-error-report")—
                                                ↓
                        doReport → pushErrorLog(去重) → pushToHmp
                                                ↓
                        adapter 内置单线程池 submit
                                                ↓
[hmp-push-1] doPush → esbReqClient.pushToEsb (Feign HTTP)
```

### **问题一：为啥使用new Thread的方式来创建新线程**

1. new Thread("hmp-error-report") 从不碰HTTP，它只做 DT0 构建 + 去重判断 + submit 到 adapter 池，微秒级结束。无论HTTP超时多久，这个线程都不会堆积。
2. 真正发 HTTP的是 adapter 的单线程池（hmp-push-1）。即使某次 HTTP 卡满60s readTimeout，也只占住那 1个线程，队列最多堵1000 条然后 Discard0ldestPolicy开始丢。线程数永远= 1，不可能00M。3. 也就是说，_new Thread的安全性根本不依赖HTTP超时_--HTTP的安全性已经由 adapter的有界池+丢弃策略兜住了。
3. 低QPS是天然保护 更关键：HTTP调用被adapter 的有界池隔离了，new Thread 不背HTTP的锅。哪怕哪天QPS 涨上去、哪怕HTTP偶尔卡住，new Thread依然微秒级结束，adapter池用DiscardoldestPolicy自我保护。两层各有各的兜底，互不拖累
4. 唯一前提是 Feign 别完全没有超时（默认 60s readTimeout 算"合理")，这个去Apollo 确认一下 feign.client.config即可。

### **问题二：为啥要有两层异步，又为啥设置成守护线程**

**为啥是两层异步**

||**第一层 new Thread（GlobalExceptionHandler）**|**第二层 adapter 有界池（TdpHmpDataPushAdapter）**|
|---|---|---|
|防什么|不让 doReport 的任何环节阻塞请求线程|不让 HTTP‑风暴 OOM 系统、不让卡住的HTTP 堵死后续推送|
|边界|Web 层 → Monitor 域|Monitor 域 → ESB/Feign HTTP|
|契约|“异常处理器必须立刻返回给用户”|“所有推送有序、有界、可丢弃”|
|去掉的后果|请求线程要承担栈遍历 + DTO 构建 + recordRepeat() 的 synchronized + flushScheduler.schedule()；未来若 pushErrorLog 加入阻塞逻辑，请求线程首当其冲|每次 pushToHmp 都得自己开启线程或同步发 HTTP；无背压、无丢弃、无串行化|

它们不是冗余，是不同威胁维度的防御：

- 第二层是安全刚需：HTTP 是真正可能长时间阻塞的环节（Feign 默认 readTimeout 60s）。没有 adapter 的有界池 + DiscardOldestPolicy，一旦 ESB 慢或挂，要么请求线程被拖、要么线程无限堆积。这层不可省。
- 第一层是请求隔离的防御纵深：doReport 里 99% 是微秒级 CPU 操作，但有两个真实的小阻塞点‑‑recordRepeat() 的 synchronized（line 568）和 flushScheduler.schedule()。低 QPS 下这些几乎无感，但第一层存在的意义是：GlobalExceptionHandler 不该假设 MonitorReportService 内部永远不阻塞。万一未来监控团队在 pushErrorLog 里加了同步落库、加了远程查询，第一层能保证请求线程不受牵连。

换句话说：第二层管“系统安全”（必须有），第一层管“请求隔离”（防御性，去掉短期内也没事但耦合变脆）。

**为啥是异步线程**

new Thread 设了 t.setDaemon(true)（line 179），adapter 池的 ThreadFactory 也设了 t.setDaemon(true)（line75）。两层都是守护线程，是同一个设计哲学：

JVM 等所有非守护线程结束才退出；守护线程会被立即强杀。

监控是 best‑effort，绝不能挡住应用生命周期：

- 如果 hmp‑error‑report 是非守护线程，且正好卡在 Feign 60s readTimeout 里，JVM 关闭时要干等最多60秒才退出
- 滚动发布时旧实例关不掉→健康检查失败→部署卡住→影响业务发版守护线程则被立即杀掉，丢失的那一条在途上报完全可接受

数据丢失的兜底在别处：去重窗口的待刷数据由 flushScheduler的 @PreDestroy shutdownDedupScheduler()(line506-525）在关闭时强制刷出，保住的是"已聚合的窗口数据"（有价值）；而单条在途HTTP（无聚合、纯瞬时）丢了就丢了。这个取舍很合理：保聚合数据、丢瞬时数据。

## 阶段 3：异常信息提取（L186-213）

doReport() 方法在异步线程中执行，提取 6 个维度的信息组装 DTO：

|维度|方法|逻辑|
|---|---|---|
|根因异常|findRootCause()|沿 getCause() 最多追溯 10 层，找到最底层异常|
|出错位置|extractCallerLocation()|遍历堆栈帧，跳过 JDK/反射代理/Spring/MyBatis/monitor 包，取第一个业务代码帧|
|行号|parseLineFromCaller()|从 ClassName.method(File.java:123) 中解析 :123 部分|
|错误码|inferErrorCode()|按异常类名关键字推断，如 nullpointer → ERR_NPE ， deadlock → ERR_DEADLOCK|
|告警摘要|buildSmartSummary()|拼接 [中文标签] 异常类名: 消息 ，截断到 480 字符|
|客户端 IP|safeGetClientIp()|依次取 X-Forwarded-For → X-Real-IP → RemoteAddr|

```c
/**
 * 从异常堆栈中提取出错位置：类名.方法名(文件名:行号)
 */
private String extractCallerLocation(Throwable t) {
    if (t == null) return "unknown";
    StackTraceElement[] stack = t.getStackTrace();
    if (stack == null || stack.length == 0) return "unknown";

    // 找第一个业务代码帧（跳过 JDK / 反射 / Spring / MyBatis 框架）
    for (StackTraceElement frame : stack) {
        String cn = frame.getClassName();
        if (cn.startsWith("java.") || cn.startsWith("javax.") || cn.startsWith("sun.")
                || cn.startsWith("jdk.")) continue;
        if (cn.contains("$Proxy") || cn.contains("CGLIB$$")) continue;
        if (cn.startsWith("org.springframework") && !cn.startsWith("org.springframework.boot")) continue;
        if (cn.startsWith("org.apache.ibatis") || cn.startsWith("org.mybatis")) continue;
        if (cn.contains(".monitor.")) continue;

        return cn + "." + frame.getMethodName() + "(" + frame.getFileName() + ":" + frame.getLineNumber() + ")";
    }
    // 兜底返回第一帧
    StackTraceElement first = stack[0];
    return first.getClassName() + "." + first.getMethodName();
}
```

### 问题一：栈帧过滤逻辑（跳过JDK/反射/Spring）

为什么这么做：异常堆栈最顶层的几帧通常是SpringAOP、CGLIB代理、MyBatis反射、JDK动态代理这些框架帧，对业务排障毫无价值("哦Spring的CglibAopProxy报错了"没人看得懂）。找到第一个业务包帧才是"我们哪行代码出问题了"。

- "如果 bug 真的在Spring框架里怎么办？" ->

答：这种少见，且即便有，框架帧也会出现在root cause的异常类型里（如org.springframework.dao.DataAccessException）

- 位置不准但类型还在"为什么org.springframework.boot 不跳过？" ->

答：boot包里有时含业务配置类，作者选择保留，是个边界判断"准确率如何？"->答：依赖包名约定，跨团队/第三方库可能误判，属于"启发式"而非精确

### 问题二：safeGetClientIp（L333‑341）-- 反代后的真实 IP 提取

```
String ip = request.getHeader("X-Forwarded-For");
if (ip == null || ip.isEmpty() || "unknown".equalsIgnoreCase(ip)) ip = request.getHeader("X-Real-IP");
if (ip == null || ip.isEmpty() || "unknown".equalsIgnoreCase(ip)) ip = request.getRemoteAddr();
if (ip != null && ip.contains(",")) ip = ip.split(",")[0].trim();
```

这是经典面试题："nginx/网关后面怎么拿客户端真实 IP？"

- X‑Forwarded‑For（XFF）：HTTP 标准代理头，多级代理时是逗号分隔列表 client, proxy1, proxy2，第一个才是真实客户端 -> 所以 split(",")[0]
- X‑Real‑IP：nginx 常用的非标头，只放一个 IP
- getRemoteAddr()：兜底，直连场景下是真实 IP，反代场景下是网关 IP（如 127.0.0.1）
- "unknown" 字符串判断：某些代理会塞字符串 "unknown" 而非 null，要单独防

可被追问的点

- "XFF 能伪造吗？" -> 能，客户端可以伪造 XFF 头，所以只信任 nginx 设置的那一跳（生产里 nginx 会覆盖客户端传来的 XFF）
- "为什么不直接 getRemoteAddr？" -> 反代后拿到的是 nginx 的 IP，没用
- 这个知识点在监控/审计/限流场景都会用到，比"组装 DTO"实用得多

## 阶段 4：推送到 MonitorReportService（L209）

```
monitorReportService.pushErrorLog(dto);
```

进入 MonitorReportService.pushErrorLog() 后还有一层 时间窗口去重聚合 （30 秒窗口内相同 errorClass + errorMethod 的异常合并为 1 条， occurCount 累加），最终通过异步线程池推送到 ESB。

## 阶段 5：返回前端（L106）

```
return Rest.failure(buildUserFriendlyMessage(ex));
```

buildUserFriendlyMessage() 把技术细节转为用户可读提示：NPE → "系统处理异常，请稍后重试"，Timeout → "请求超时，请稍后重试"，其他截断原始消息到 100 字符。

## 完整数据流

```
Controller 抛出异常
  → Spring 匹配到 @ExceptionHandler(Exception.class)
  → handleSystemException()
      ├─ log.error 打日志
      ├─ isSqlRelatedException() 过滤SQL重复
      ├─ reportErrorAsync() ──→ new Thread("hmp-error-report")
      │                          → doReport()
      │                              ├─ findRootCause()        找根因
      │                              ├─ extractCallerLocation() 找业务代码帧
      │                              ├─ inferErrorCode()       推断错误码
      │                              ├─ buildSmartSummary()    拼摘要
      │                              └─ monitorReportService.pushErrorLog(dto)
      │                                  → 30s窗口去重聚合
      │                                  → 异步线程池推送ESB
      │                                  → HMP平台 → 飞书告警
      └─ return Rest.failure(友好提示) → 前端
```