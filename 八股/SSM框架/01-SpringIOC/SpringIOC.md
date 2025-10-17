---
tags:
  - 八股
  - SSM
  - Spring
  - IOC
status: 待复习
created: 2026-05-02
---

# Spring IOC/DI 与循环依赖

> [!summary] 我的速记
> **IOC = 控制反转，DI = 依赖注入。** 原来你自己 new 对象 → 现在 Spring 帮你创建并注入。容器就是个大 Map<beanName, beanObject>。
> **循环依赖靠三级缓存解决：** singletonFactories（刚创建未填充的早期引用）→ earlySingletonObjects（提前暴露的半成品）→ singletonObjects（成品）。
> **@Autowired 底层：** 先按 type 找（byType）→ 找到多个同类型则按 name（byName）→ 都不行则 @Qualifier 指定。

---

## 一、什么是 IOC/DI？

| 概念 | 一句话 | 类比 |
|------|--------|------|
| **IOC** | 把创建对象的权力交给容器 | 以前你买菜做饭 → 现在外卖平台帮你搞定 |
| **DI** | 容器自动把依赖的对象注入进来 | Service 需要 Mapper → Spring 自动塞进来 |

```java
// 没有 IOC：自己管理依赖
public class OrderService {
    private OrderMapper mapper = new OrderMapperImpl();  // 自己 new，紧耦合
}

// 有了 IOC：Spring 管理
@Service
public class OrderService {
    @Autowired
    private OrderMapper mapper;  // Spring 注入，松耦合，方便测试和切换实现
}
```

---

## 二、IOC 容器——BeanFactory vs ApplicationContext

| | BeanFactory | ApplicationContext |
|---|---|---|
| 定位 | 底层基础接口 | 高级容器（继承 BeanFactory） |
| Bean 创建 | **延迟加载**（用到才创建） | **启动时全部预加载**（singleton 默认） |
| 功能 | 基本 DI | +AOP、事件、国际化、资源加载 |
| 实际使用 | 极少直接用 | ✅ ApplicationContext（就是常说的 Spring 容器） |

> 🧠 **面试话术：** "Spring 底层是 BeanFactory，但我们平时用的 ApplicationContext 是它的子接口，多了 AOP、国际化、事件发布等高级功能。默认启动时就把所有 singleton Bean 创建好——这是为了启动时就暴露配置错误，而不是等到运行业务才发现。"

---

## 三、DI 三种注入方式

| 方式 | 代码 | 推荐度 | 原因 |
|------|------|--------|------|
| **构造器注入** | `public Service(Dao dao) { this.dao = dao; }` | ✅ 推荐 | 不可变、可测试、强制依赖 |
| **Setter 注入** | `public setDao(Dao dao)` | 可选用 | 可选依赖 |
| **字段注入** | `@Autowired private Dao dao` | ❌ 不推荐 | 反射破坏封装、难测试、隐藏依赖 |

> 🧠 **Spring 官方推荐构造器注入：** 依赖不可变（final）+ 不依赖反射 + 单元测试可显式传 Mock。

---

## 四、Bean 作用域

| 作用域 | 说明 | 生命周期 |
|--------|------|----------|
| **singleton** | 整个容器只有一个实例 | ✅ 默认，容器启停 |
| **prototype** | 每次获取都创建新实例 | 创建后容器不管销毁 |
| **request** | 每个 HTTP 请求一个实例 | Web 环境，请求结束销毁 |
| **session** | 每个 HTTP Session 一个实例 | Web 环境，Session 结束销毁 |

> 🧠 **singleton Bean 里注入 prototype Bean → prototype 失效！** 因为 singleton 只创建一次，注入的 prototype 也只被注入一次。解决方法：`@Lookup` 或注入 `ObjectFactory`。

---

## 五、循环依赖与三级缓存——面试最爱问

### 5.1 问题：A 依赖 B，B 依赖 A → 怎么解？

```java
@Service
public class A {
    @Autowired private B b;  // A 需要 B
}
@Service
public class B {
    @Autowired private A a;  // B 需要 A → 循环！
}
```

### 5.2 Spring 解法：三级缓存

```
singletonObjects（一级缓存）
  → 存完全创建好的 Bean

earlySingletonObjects（二级缓存）
  → 存提前暴露的早期 Bean（已实例化但未填充属性）

singletonFactories（三级缓存）
  → 存 Bean 的 ObjectFactory（创建早期引用的工厂方法）
```

**解决流程：**

```
① 创建 A → 实例化 A（构造方法调用完成，属性还没填）
   → 把 A 的 ObjectFactory 放入三级缓存（"我可以提前给你 A 的引用"）
   → 开始填充 A 的属性 → 发现需要 B

② 创建 B → 实例化 B
   → 把 B 的 ObjectFactory 放入三级缓存
   → 填充 B 的属性 → 发现需要 A
   → 从三级缓存找到 A 的 ObjectFactory → 调用它获得 A 的早期引用
   → B 注入 A 成功 → B 创建完成 → 放入一级缓存

③ 回到 A → 从一级缓存获取完整的 B → 注入 → A 创建完成
```

> 🧠 **为什么不用二级缓存？** 如果只有二级缓存，当 A 需要经过 AOP 代理时，拿到的早期引用是原始对象，而后面的 AOP 后置处理器会生成代理对象——会导致 B 拿到的 A 和最终暴露的 A 不一样。三级缓存的 ObjectFactory 可以在需要时动态决定返回原始对象还是代理对象。

### 5.3 无法解决的循环依赖

| 情况 | 能解决？ | 原因 |
|------|---------|------|
| 两个 singleton 的 setter 注入 | ✅ 能 | 三级缓存 |
| 两个 singleton 的**构造器注入** | ❌ 不能 | 构造器需要对方已经创建好 → 死锁。**只能用 @Lazy + 构造器注入** |
| prototype 的循环依赖 | ❌ 不能 | 每次创建新对象，三级缓存不生效 |

---

## 常见面试问题清单

| #   | 问题                                     | 回答要点                                                                                                       |
| --- | -------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| Q1  | **IOC 和 DI 是什么？**                      | IOC=控制反转(Spring管理对象创建)，DI=依赖注入(自动注入)。核心：解耦                                                                 |
| Q2  | **BeanFactory vs ApplicationContext？** | ApplicationContext 是 BeanFactory 扩展——多了AOP/事件/国际化。默认启动预加载 singleton                                        |
| Q3  | **@Autowired 注入原理？**                   | DefaultListableBeanFactory 中 byType → 多个同类型则 byName → @Qualifier 精确指定                                      |
| Q4  | **循环依赖怎么解决？**                          | 三级缓存：singletonObjects(成品)→earlySingletonObjects(半成品)→singletonFactories(工厂)。ObjectFactory 可动态决定是否返回 AOP 代理 |
| Q5  | **什么情况三级缓存也解决不了？**                     | 构造器注入的循环依赖（死锁）、prototype 的循环依赖                                                                             |
| Q6  | **singleton 中注入 prototype 的问题？**       | prototype 只注入一次，后续不变。用 @Lookup 或 ObjectFactory 解决                                                          |

---

## 面试满分话术

> **面试官：Spring IOC 和循环依赖怎么解决？**

> IOC 就是 Spring 通过容器管理 Bean 的创建和依赖注入，解放开发者手动 new 对象的繁琐。DI 有三种方式——构造器注入（官方推荐，不可变）、Setter 注入、@Autowired 字段注入。
>
> 循环依赖是 Spring 面试经典题。Spring 用三级缓存解决 singleton 的 setter 循环依赖。创建 A 时，实例化后但未注入属性前，把 A 的 ObjectFactory 放入三级缓存；然后去创建依赖的 B，B 需要 A 时从三级缓存拿到 A 的早期引用；B 创建好后，A 注入 B 完成。三级缓存之所以要存在，是因为 AOP——如果 A 需要被代理，ObjectFactory 可以在返回引用时动态决定返回原始对象还是代理对象。
>
> 但构造器循环依赖三级缓存也解不了，因为构造器阶段还没形成早期引用就死锁了。prototype 也不支持，因为每次 new 新对象，缓存没用。

---

## 跨题串联

| 引到 | 衔接 |
|------|------|
| [[../../JVM/02-类加载机制/类加载机制\|类加载机制]] | 「IOC 容器本质上就是通过反射(Class.forName)创建 Bean 实例——反射是 IOC 的底层基础」 |
| [[../02-SpringAOP/SpringAOP\|Spring AOP]] | 「三级缓存 ObjectFactory 的核心作用就是应对 AOP 代理——同一个 Bean 的原始对象和代理对象必须一致」 |
| [[../03-Bean生命周期/Bean生命周期\|Bean 生命周期]] | 「三级缓存中 Bean 在哪个阶段被放入？实例化(构造)完成后、属性填充前——对应生命周期的 instantiate→populate 之间」 |

---

## 记忆口诀

```
IOC 反转交容器，DI 自动注入来解耦。
singleton 单例全局用，prototype 每次新创建。
循环依赖用三级，singletonFactory 存早期。
构造器注入最推荐，final 不可变又安全。
```
