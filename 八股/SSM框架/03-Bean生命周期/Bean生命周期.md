---
tags:
  - 八股
  - SSM
  - Spring
  - Bean生命周期
status: 待复习
created: 2026-05-02
---

# Spring Bean 生命周期

> [!summary] 我的速记
> **Bean 生命周期 4 大阶段：实例化 → 属性填充 → 初始化 → 销毁。**
> **面试重点是初始化阶段的回调顺序：@PostConstruct → InitializingBean → init-method。**
> **AOP 代理在哪生成？** BeanPostProcessor.after（初始化后处理器）→ 这是生成代理对象的时机，也是三级缓存放 ObjectFactory 的原因。

---

## 一、完整生命周期（面试必画）
![](https://my.feishu.cn/space/api/box/stream/download/asynccode/?code=MDEyNDg0OTE1NTVlY2U5NmNhMzgyZTgxNmIyMDc3OTFfVDJPRWZhOUs5VlBocGFvYk9yQ3lueWFIWXpOc0pab29fVG9rZW46UDV5c2JXRThob1hENmZ4dUgyUmNRQ0E4bjBmXzE3Nzc3OTA5NTM6MTc3Nzc5NDU1M19WNA)

```
┌─ 实例化（Instantiate）
│    构造方法调用，Bean 对象创建（属性为默认值）
│
├─ 属性填充（Populate）
│    依赖注入（@Autowired 注入）
│    BeanNameAware.setBeanName()
│    BeanFactoryAware.setBeanFactory()
│    ApplicationContextAware.setApplicationContext()
│
├─ 初始化（Initialize）
│    BeanPostProcessor.postProcessBeforeInitialization()  ← 前置处理
│    @PostConstruct 注解方法
│    InitializingBean.afterPropertiesSet()
│    init-method（XML 或 @Bean(initMethod=)）
│    BeanPostProcessor.postProcessAfterInitialization()   ← 后置处理 → AOP 代理生成！
│    ↑ 返回代理对象（如果有 AOP）替代原始对象放入容器
│
├─ 就绪（Ready）
│    Bean 可用
│
└─ 销毁（Destroy）
      @PreDestroy 注解方法
      DisposableBean.destroy()
      destroy-method（XML 或 @Bean(destroyMethod=)）
```

> 🧠 **AOP 代理在 postProcessAfterInitialization 生成**——这就是为什么三级缓存不能只存原始 Bean：如果 Bean 需要被 AOP 代理，其他依赖它的 Bean 必须拿到代理对象而非原始对象。

---

## 二、三个初始化回调的顺序（面试高频）

```java
@Component
public class MyBean implements InitializingBean, DisposableBean {

    @PostConstruct
    public void postConstruct() {
        System.out.println("1. @PostConstruct");          // 最先
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("2. InitializingBean");        // 第二
    }

    @Bean(initMethod = "customInit")
    public void customInit() {
        System.out.println("3. init-method");             // 最后
    }
}
```

| 顺序 | 回调 | 方式 | 特点 |
|------|------|------|------|
| ① | `@PostConstruct` | 注解（JSR-250） | ✅ 最推荐——简洁、解耦 Spring |
| ② | `afterPropertiesSet()` | 实现 InitializingBean 接口 | ❌ 不推荐——耦合 Spring |
| ③ | `init-method` | XML 或 @Bean 属性 | 可选用 |

> 🧠 **为什么 @PostConstruct 最推荐？** 它是 Java 标准注解（不是 Spring 专有），代码不耦合 Spring 框架。

---

## 三、BeanPostProcessor——Spring 最强大的扩展点

```java
public interface BeanPostProcessor {
    // 初始化前调用
    default Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;  // 可以返回原 Bean 或包装后的 Bean
    }

    // 初始化后调用（AOP 代理生成点！）
    default Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;
    }
}
```

| 应用 | 哪个 Processor | 做了什么 |
|------|---------------|----------|
| **@Autowired** | AutowiredAnnotationBeanPostProcessor | 属性填充时注入依赖 |
| **@PostConstruct** | CommonAnnotationBeanPostProcessor | 初始化阶段调用 @PostConstruct 方法 |
| **AOP 代理** | AbstractAutoProxyCreator | postProcessAfterInitialization 中创建代理 |
| **@Transactional** | InfrastructureAdvisorAutoProxyCreator | 为 @Transactional 的 Bean 生成事务代理 |
| **@Async** | AsyncAnnotationBeanPostProcessor | 为 @Async 方法生成异步代理 |

---

## 四、BeanFactoryPostProcessor vs BeanPostProcessor

| | BeanPostProcessor | BeanFactoryPostProcessor |
|---|---|---|
| 操作对象 | **Bean 实例** | **BeanDefinition**（Bean 的元数据/定义） |
| 阶段 | 每个 Bean 初始化前后 | 所有 BeanDefinition 加载后、实例化前 |
| 典型应用 | @Autowired、AOP、@PostConstruct | `PropertySourcesPlaceholderConfigurer`（解析 `${}` 占位符） |

> 🧠 **时间线：** BeanFactoryPostProcessor → 实例化 Bean → BeanPostProcessor。前者在"结婚登记"阶段改信息，后者在"婚后"阶段改行为。

---

## 常见面试问题清单

| # | 问题 | 回答要点 |
|---|------|----------|
| Q1 | **Bean 生命周期有哪些阶段？** | 实例化→属性填充→初始化(3个回调有顺序)→就绪→销毁 |
| Q2 | **三个初始化回调顺序？** | @PostConstruct → InitializingBean → init-method。推荐 @PostConstruct |
| Q3 | **AOP 代理在哪个阶段生成？** | BeanPostProcessor.postProcessAfterInitialization()——初始化后、放入容器前 |
| Q4 | **BeanPostProcessor 做什么？** | 对 Bean 实例做后处理——@Autowired、@PostConstruct、AOP 代理生成 |
| Q5 | **BeanFactoryPostProcessor vs BeanPostProcessor？** | 前者改 BeanDefinition(元数据)、后者改 Bean 实例。时机不同、对象不同 |
| Q6 | **Bean 如何感知到容器？** | 实现 Aware 接口——BeanNameAware/BeanFactoryAware/ApplicationContextAware |

---

## 面试满分话术

> **面试官：讲讲 Spring Bean 的完整生命周期？**
>  **实例化环节**：Spring 通过反射创建 Bean 对象，若构造器有依赖，会顺带注入依赖，如同建好毛坯房。
- **属性注入环节**：完成依赖注入操作，如同对毛坯房进行装修。
- **初始化环节**：该环节为面试加分核心，呈三明治结构：先执行 Aware 接口方法，让 Bean 知晓自身身份与所在工厂；再执行 BeanPostProcessor 的 before 方法；接着执行 Bean 自身的初始化逻辑，如 @PostConstruct、afterPropertiesSet 等；最后执行 BeanPostProcessor 的 after 方法，AOP 动态代理多在此完成，最终拿到的可能是代理对象。
- **销毁环节**：容器关闭时，执行 @PreDestroy、DisposableBean 的 destroy 等方法，释放资源完成收尾。

> Bean 生命周期分四大阶段。**实例化**——调用构造方法创建 Bean 对象。**属性填充**——@Autowired 注入依赖，同时 Aware 接口回调让 Bean 感知容器。**初始化**是面试重点——按顺序执行 BeanPostProcessor 前置处理 → @PostConstruct → InitializingBean.afterPropertiesSet → init-method → BeanPostProcessor 后置处理。AOP 代理就是在后置处理阶段生成的。
>
> **销毁阶段**——容器关闭时按 @PreDestroy → DisposableBean.destroy → destroy-method 的顺序执行清理。BeanPostProcessor 是 Spring 最核心的扩展点——@Autowired、@PostConstruct、AOP、@Transactional 都是通过它实现的。

---

## 记忆口诀

```
实例化创建空壳，属性填充注依赖。
Aware 感知容器，初始化三步走。
PostConstruct 最先，afterPropertiesSet。
init-method 最后，后置处理生代理。
容器关闭销毁它，PreDestroy 先执行。
```
