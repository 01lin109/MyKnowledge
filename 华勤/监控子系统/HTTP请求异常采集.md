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