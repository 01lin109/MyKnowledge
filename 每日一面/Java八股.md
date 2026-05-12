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