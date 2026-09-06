# Spring AOP

## 1. Spring AOP 动态代理机制

### JDK 动态代理 vs CGLIB 动态代理

| 特性 | JDK 动态代理 | CGLIB 动态代理 |
|------|-------------|---------------|
| 实现原理 | 基于 **Java 接口**。通过 `Proxy.newProxyInstance()` 创建。 | 基于 **继承（子类）**。通过 ASM 字节码技术生成目标类的子类。 |
| 限制条件 | 目标类 **必须实现至少一个接口**。 | 目标类不需要实现接口，但类和方法 **不能被 `final` 修饰**。 |
| Spring 选择策略 | 如果目标对象实现了接口，默认使用 JDK 代理。 | 如果目标对象没有实现接口，则强制使用 CGLIB。 |

> **注意**：Spring Boot 2.x 之后默认强制使用 CGLIB 构建代理。

## 2. 同类调用 AOP 拦截失效的原因

**拦截不能成功触发。**

### 原理分析

AOP 的本质是通过**代理对象**在调用目标方法前后织入逻辑（比如反射填充字段）。

- 当你从外部调用 `service.methodA()` 时，调用的是 **Proxy（代理对象）**，拦截生效。
- 但是在 `methodA` 内部调用 `this.save()` 时，这里的 `this` 是 **Target（原始目标对象）**。由于原始对象的方法内部没有代理逻辑，因此 AOP 拦截器（Interceptor）根本不会被触发。

## 3. 解决方案

### 方法一：通过 `AopContext` 获取当前代理对象（最常用）

这是 Spring 官方提供的"自救"方案。

1. 在启动类或配置类开启：`@EnableAspectJAutoProxy(exposeProxy = true)`。
2. 在代码中使用 `AopContext.currentProxy()` 强转为当前接口/类再进行调用：

```java
public void methodA() {
    // 这样调用 save，本质上是代理对象在调用，拦截会生效
    ((YourService) AopContext.currentProxy()).save();
}
```

### 方法二：利用 Spring 注入自己（Self-Injection）

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

### 方法三：拆分到另一个 Service 类

将需要被拦截的方法拆分到另一个 Service 类中，通过依赖注入调用。

## 4. 反射性能优化

### 优化手段

1. **缓存（Caching）**：反射最耗时的步骤在于查找类的方法和属性（如 `getDeclaredMethod`）。在组件初始化时，针对涉及到的 Service 方法或实体类，将解析出来的 `Method` 或 `Field` 对象缓存到 `ConcurrentHashMap` 中，后续直接复用，避免重复查找。

2. **关闭安全检查**：在获取到 `Method` 或 `Field` 后，通过调用 `setAccessible(true)` 可以跳过 Java 的访问控制检查，这能带来显著的性能提升。

3. **替代方案引入**：如果性能要求极高，可以考虑放弃原生反射，引入高性能的字节码框架（如 `ReflectASM`）或者使用 `MethodHandles`（Java 7 引入的轻量级方法指针），它们在底层使用字节码生成，执行速度几乎等同于直接调用代码。
