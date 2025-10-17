---
tags:
  - 八股
  - SSM
  - Spring
  - AOP
status: 待复习
created: 2026-05-02
---

# Spring AOP 面向切面编程

> [!summary] 我的速记
> **AOP = 在不改原代码的前提下，给方法横向加功能（日志/事务/权限）。底层是动态代理。**
> **JDK 动态代理：必须基于接口，创建接口实现类的代理对象。**
> **CGLIB 代理：无需接口，继承目标类生成子类，final 类/方法不行。**
> **Spring 默认策略：有接口用 JDK，无接口用 CGLIB。Spring Boot 2.0 起默认都用 CGLIB。**

---

## 一、AOP 核心概念——五要素

```
切面（Aspect）= 切点 + 通知
  切点（Pointcut）= 在哪里切（execution(* com.example..*.*(..))）
  通知（Advice）  = 切了之后做什么
    ├── @Before        → 方法执行前
    ├── @AfterReturning → 正常返回后
    ├── @AfterThrowing  → 异常抛出后
    ├── @After          → 无论正常还是异常都执行
    └── @Around         → 环绕（最强，可控制方法是否执行）

连接点（JoinPoint）= 所有可能被切入的点（每个方法都是）
织入（Weaving）= 把切面应用到目标对象并生成代理的过程
```

> 🧠 **一句话：** 切面 = 在哪里(切点) + 做什么(通知)。Spring AOP 的底层是动态代理，在 Bean 后置处理时织入。

---

## 二、JDK 动态代理 vs CGLIB

|               | JDK 动态代理                   | CGLIB 代理                      |
| ------------- | -------------------------- | ----------------------------- |
| **原理**        | 实现同一个接口                    | 继承目标类（生成子类）                   |
| **要求**        | **必须有接口**                  | 不需要接口                         |
| **限制**        | 只能代理接口方法                   | `final` 类/方法不能被代理             |
| **本质**        | `Proxy.newProxyInstance()` | ASM 字节码生成子类                   |
| **性能**        | 创建快，执行略慢（反射 invoke）        | 创建慢（生成字节码），执行快                |
| **Spring 默认** | 旧版：有接口用 JDK                | **Spring Boot 2.0+ 默认 CGLIB** |

```java
// JDK 动态代理
public class JdkProxy implements InvocationHandler {
    private Object target;

    public Object getProxy(Object target) {
        this.target = target;
        return Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),  // ← 必须传接口数组
            this
        );
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("前置通知");        // 切面逻辑
        Object result = method.invoke(target, args);  // 反射调用目标方法
        System.out.println("后置通知");
        return result;
    }
}

// CGLIB 代理
public class CglibProxy implements MethodInterceptor {
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("前置通知");
        Object result = proxy.invokeSuper(obj, args);  // 调用父类方法
        System.out.println("后置通知");
        return result;
    }
}
```

> 🧠 **为什么 Spring Boot 2.0+ 默认 CGLIB？** 直接代理类更灵活，不需要强制接口。而且 CGLIB 3.0 后性能已不差。

---

## 三、五种通知类型

```java
@Aspect
@Component
public class LogAspect {

    @Pointcut("execution(* com.example.service.*.*(..))")
    public void servicePointcut() {}

    @Before("servicePointcut()")
    public void before(JoinPoint jp) {          // 执行前
        System.out.println("开始: " + jp.getSignature().getName());
    }

    @AfterReturning(value = "servicePointcut()", returning = "result")
    public void afterReturning(Object result) { // 正常返回
        System.out.println("返回: " + result);
    }

    @AfterThrowing(value = "servicePointcut()", throwing = "e")
    public void afterThrowing(Exception e) {     // 异常
        System.out.println("异常: " + e.getMessage());
    }

    @After("servicePointcut()")
    public void after() {                        // 最终（类似 finally）
        System.out.println("结束");
    }

    @Around("servicePointcut()")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {  // 环绕
        System.out.println("环绕-前");
        Object result = pjp.proceed();  // ← 手动调用目标方法
        System.out.println("环绕-后");
        return result;
    }
}
```

> 🧠 **@Around 最强大但也最危险：** 必须手动调 `pjp.proceed()`，忘了调用 → 目标方法永远不执行。

---

## 四、AOP 的实际应用场景

| 场景        | 实现                           | 原理                          |
| --------- | ---------------------------- | --------------------------- |
| **声明式事务** | `@Transactional`             | 事务拦截器通过 AOP 在方法前后开启/提交/回滚事务 |
| **日志记录**  | 自定义 `@Log` 注解 + AOP          | 切面记录入参/出参/耗时                |
| **权限校验**  | `@PreAuthorize`              | 方法执行前校验权限，不通过则抛异常           |
| **缓存**    | `@Cacheable` / `@CacheEvict` | 方法执行前查缓存、执行后写缓存             |
| **重试**    | `@Retryable`                 | 异常时自动重试                     |

---

## 五、AOP 失效场景

| 场景              | 为什么失效                         | 解决                                  |
| --------------- | ----------------------------- | ----------------------------------- |
| **自调用**         | `this.methodB()` 不走代理 → 切面不生效 | 注入自己 → `self.methodB()`；或拆到另一个 Bean |
| **非 public 方法** | Spring 默认只代理 public           | 配 AspectJ（编译期织入）                    |
| **final 类/方法**  | CGLIB 基于继承，final 不能覆盖         | 不用 final / 改用 JDK 代理                |
| **同类事务失效**      | 同上——`@Transactional` 本质是 AOP  | 同上                                  |
|                 |                               |                                     |

```java
// ❌ 自调用导致事务失效
@Service
public class UserService {
    public void outer() {
        this.inner();  // ← this.inner() 走的是原始对象，不是代理 → 事务失效！
    }

    @Transactional
    public void inner() { ... }
}
```

> 🧠 **自调用是 Spring AOP 最常见的坑——因为 AOP 是通过代理对象生效的，this 指向原始对象。**

---

## 常见面试问题清单

| #   | 问题                             | 回答要点                                                                                   |
| --- | ------------------------------ | -------------------------------------------------------------------------------------- |
| Q1  | **AOP 是什么？底层原理？**              | 面向切面编程。底层 JDK 动态代理（接口）或 CGLIB（子类）。在方法前后插入逻辑                                            |
| Q2  | **JDK 代理 vs CGLIB？**           | JDK 需接口、反射 invoke；CGLIB 无需接口、继承+ASM 生成子类。Spring Boot 2.0+ 默认 CGLIB                     |
| Q3  | **五种通知执行顺序？**                  | @Around前 → @Before → 方法执行 → @Around后 → @After → @AfterReturning(正常)/@AfterThrowing(异常) |
| Q4  | **AOP 有哪些应用场景？**               | 声明式事务(@Transactional)、日志、权限(@PreAuthorize)、缓存(@Cacheable)、重试                           |
| Q5  | **AOP 什么时候失效？**                | 自调用(this.xxx)、非public、final类/方法、同类事务                                                   |
| Q6  | **@Around 忘了调 proceed() 会怎样？** | 目标方法永远不会执行，且不会有任何报错                                                                    |

---

## 面试满分话术

> **面试官：AOP 的底层原理和使用场景？**

> AOP 是面向切面编程，在不修改原代码的前提下给方法横向加功能。Spring AOP 底层用动态代理：有接口时用 JDK 动态代理（Proxy.newProxyInstance），没接口时用 CGLIB（ASM 生成子类）。Spring Boot 2.0 起默认都走 CGLIB。
>
> AOP 最经典的落地是 @Transactional——事务拦截器在方法执行前开启事务，正常返回则提交，异常则回滚。其他应用包括日志、权限、缓存等。
>
> AOP 最常见的坑是自调用——同类中一个方法调另一个方法，走的是 this 而不是代理对象 → 切面不生效。解决方案是注入自己（self.methodB()）或拆到另一个 Bean。

---

## 记忆口诀

```
AOP 切面横插入，代理对象帮你忙。
JDK 基于接口做，CGLIB 继承来增强。
通知五种有先后，Around 最强别忘调。
事务日志和权限，全靠切面来加持。
自调用是大陷阱，this 不走代理对象。
```
