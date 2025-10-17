---
tags:
  - 八股
  - SSM
  - Spring
status: 待复习
created: 2026-05-02
---

# SSM 框架总览

> [!summary] 模块速记
> **SSM 三大框架各管一摊：Spring 管对象（IOC/AOP/事务）、SpringMVC 管请求（DispatcherServlet 调度链）、MyBatis 管数据库（ORM 映射）。**
> **一条主线：HTTP请求 → SpringMVC DispatchServlet → Controller → Service(Spring管理,@Transactional事务) → MyBatis Mapper → SQL执行 → 结果返回。**

---

## 模块知识地图

```
SSM 框架
  │
  ├── 01. Spring IOC/DI
  │     ├── IOC 容器：BeanFactory vs ApplicationContext
  │     ├── DI 方式：构造器注入、Setter注入、@Autowired
  │     ├── Bean 作用域：singleton、prototype、request、session
  │     └── 循环依赖 → 三级缓存（singletonFactories/early/单例池）
  │
  ├── 02. Spring AOP
  │     ├── 代理模式：JDK 动态代理（接口） vs CGLIB（子类）
  │     ├── AOP 术语：切面、切点、通知、连接点
  │     ├── 五种通知：@Before/@After/@Around/@AfterReturning/@AfterThrowing
  │     └── AOP 应用场景：事务、日志、权限、缓存
  │
  ├── 03. Bean 生命周期
  │     ├── 实例化 → 属性填充 → BeanNameAware → BeanFactoryAware
  │     ├── → BeanPostProcessor.before → @PostConstruct
  │     ├── → InitializingBean → init-method
  │     ├── → BeanPostProcessor.after → AOP 代理生成
  │     └── → 就绪 → @PreDestroy → DisposableBean → destroy-method
  │
  ├── 04. Spring 事务
  │     ├── 声明式事务 @Transactional vs 编程式事务
  │     ├── 事务传播行为（7种）：REQUIRED/REQUIRES_NEW/NESTED
  │     ├── 事务隔离级别 + 脏读/不可重复读/幻读
  │     └── 失效场景：自调用、非public、捕获异常、多线程
  │
  ├── 05. SpringMVC
  │     ├── DispatcherServlet 请求处理流程
  │     ├── 核心组件：HandlerMapping/HandlerAdapter/ViewResolver
  │     ├── @Controller vs @RestController
  │     └── 拦截器 vs 过滤器
  │
  └── 06. MyBatis
        ├── 执行流程：SqlSession → Executor → StatementHandler → JDBC
        ├── #{} vs ${}（预编译 vs 字符串替换，SQL注入）
        ├── 一级缓存（SqlSession） vs 二级缓存（Mapper）
        └── 插件机制（Interceptor + 动态代理）
```

---

## 模块进度

| 序号 | 主题 | 状态 | 笔记数 |
|------|------|------|--------|
| 01 | Spring IOC/DI | ⬜ 待复习 | 1 |
| 02 | Spring AOP | ⬜ 待复习 | 1 |
| 03 | Bean 生命周期 | ⬜ 待复习 | 1 |
| 04 | Spring 事务 | ⬜ 待复习 | 1 |
| 05 | SpringMVC | ⬜ 待复习 | 1 |
| 06 | MyBatis | ⬜ 待复习 | 1 |
