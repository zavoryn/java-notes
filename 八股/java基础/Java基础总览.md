---
tags:
  - 八股
  - java基础
  - 复习
created: 2026-04-29
reviewed: 2026-04-30
status: 已完成
---

# Java基础 总览

> **昨日已完成（4/29）**：泛型、finally 中 return、重写重载、== vs equals、hashCode 契约、String 三兄弟、抽象类 vs 接口、面向对象三大特性
> **今日目标（4/30）**：补完所有遗漏高频考点，完成 Java 基础模块复习

---

## 一、完整知识体系

```
Java 基础
│
├── 📦 一、基本语法
│   ├── 数据类型
│   │   ├── 8 种基本类型 (byte/short/int/long/float/double/char/boolean)
│   │   ├── 引用类型
│   │   ├── 🔴 自动装箱/拆箱 + 包装类缓存池
│   │   └── 类型转换（隐式 & 强制）
│   ├── 运算符
│   │   ├── 算术 / 关系 / 逻辑 / 位运算
│   │   ├── 🔴 短路运算 (&& vs &, || vs |)
│   │   └── 三元运算符
│   ├── 流程控制 (if/switch/for/while)
│   └── 🔴 关键字深度 (static/final/this/super)
│
├── 🧱 二、面向对象（核心中的核心）
│   ├── ✅ 三大特性 — 封装/继承/多态
│   ├── ✅ 重写 vs 重载
│   ├── ✅ 抽象类 vs 接口 (JDK8+)
│   ├── 🔴 内部类
│   │   ├── 成员内部类
│   │   ├── 静态内部类
│   │   ├── 局部内部类
│   │   └── 匿名内部类
│   └── 🔴 权限修饰符 (private/default/protected/public)
│
├── 🔧 三、核心机制
│   ├── ✅ 泛型（类型擦除/通配符/影响）
│   ├── 🔥 反射（重中之重）
│   │   ├── Class 对象获取方式
│   │   ├── 动态创建对象/调用方法/访问字段
│   │   └── 应用场景（框架、动态代理）
│   ├── 🔴 注解
│   │   ├── 元注解 (@Target/@Retention 等)
│   │   ├── 自定义注解
│   │   └── 注解处理器
│   ├── 🔥 异常处理
│   │   ├── Throwable 体系 (Error / Exception)
│   │   ├── checked vs unchecked 异常
│   │   ├── try-catch-finally 执行顺序
│   │   ├── ✅ finally 中 return 的行为
│   │   └── 🔴 try-with-resources
│   └── 🔴 序列化
│       ├── Serializable 接口
│       ├── serialVersionUID 作用
│       └── transient 关键字
│
├── 📝 四、字符串 & 常用类
│   ├── ✅ String / StringBuffer / StringBuilder
│   ├── ✅ == vs equals
│   ├── ✅ hashCode & equals 契约
│   ├── 🔥 Object 类方法
│   │   ├── getClass / hashCode / equals / clone
│   │   ├── toString / notify / notifyAll / wait
│   │   └── finalize
│   └── 🔥 深拷贝 vs 浅拷贝
│       ├── Cloneable 接口
│       └── 序列化实现深拷贝
│
├── 🧵 五、多线程基础（大纲→见 [[../多线程/多线程总览|多线程与并发编程]]）
│   ├── 🔥 线程创建方式
│   │   ├── Thread 类
│   │   ├── Runnable 接口
│   │   └── 🔴 Runnable vs Callable + FutureTask
│   ├── 🔴 线程生命周期（6 种状态）
│   └── 🔴 基础同步 (synchronized / volatile / wait-notify)
│
├── 📚 六、集合框架基础（大纲→见 [[../Java集合/]]）
│   ├── Collection 体系
│   │   ├── List (ArrayList / LinkedList / Vector)
│   │   ├── Set (HashSet / TreeSet / LinkedHashSet)
│   │   └── Queue (LinkedList / PriorityQueue)
│   └── Map 体系
│       ├── HashMap (put流程/扩容/寻址)
│       ├── ConcurrentHashMap (1.7分段锁 vs 1.8 CAS)
│       ├── TreeMap / LinkedHashMap
│       └── Hashtable
│
├── 🔄 七、I/O 流
│   ├── 字节流 (InputStream/OutputStream)
│   ├── 字符流 (Reader/Writer)
│   ├── 缓冲流 / 转换流
│   ├── BIO / NIO / AIO 区别
│   └── NIO 三大核心 (Channel/Buffer/Selector)
│
├── 🎯 八、JDK 新特性
│   ├── JDK8: Lambda / Stream / Optional
│   ├── JDK8: 新的日期时间 API
│   ├── JDK9-17: var / record / sealed class / 文本块
│   └── JDK17/21 LTS 核心更新
│
└── ⚡ 九、其他高频考点
    ├── 🔥 值传递 (Java 只有值传递，没有引用传递)
    ├── 🔥 动态代理 (JDK 动态代理 vs CGLIB)
    ├── 🔴 SPI 机制
    ├── 🔴 语法糖 (switch String / 泛型擦除 / 自动拆装箱等)
    └── 🔴 BigDecimal 精度问题
```

---

## 二、复习进度追踪

| 序号 | 主题 | 状态 | 优先级 |
|------|------|------|--------|
| 1 | 泛型 | ✅ 已完成 | 🔥 必会 |
| 2 | finally 中 return | ✅ 已完成 | 🔥 必会 |
| 3 | 重写 vs 重载 | ✅ 已完成 | 🔥 必会 |
| 4 | == vs equals | ✅ 已完成 | 🔥 必会 |
| 5 | hashCode & equals 契约 | ✅ 已完成 | 🔥 必会 |
| 6 | String/StringBuffer/StringBuilder | ✅ 已完成 | 🔥 必会 |
| 7 | 抽象类 vs 接口 | ✅ 已完成 | 🔥 必会 |
| 8 | 面向对象三大特性 | ✅ 已完成 | 🔥 必会 |
| 9 | Runnable vs Callable | ⬜ 待复习 | 🔥 必会 |
| 10 | 反射 | ⬜ 待复习 | 🔥 必会 |
| 11 | 异常处理（全体系） | ⬜ 待复习 | 🔥 必会 |
| 12 | 包装类 + 缓存池 | ⬜ 待复习 | 🔥 必会 |
| 13 | Object 类方法 | ⬜ 待复习 | 🔥 必会 |
| 14 | 值传递 | ⬜ 待复习 | 🔥 必会 |
| 15 | 深拷贝 vs 浅拷贝 | ⬜ 待复习 | 📌 重要 |
| 16 | 内部类（4 种） | ⬜ 待复习 | 📌 重要 |
| 17 | 注解 | ⬜ 待复习 | 📌 重要 |
| 18 | 序列化 | ⬜ 待复习 | 📌 重要 |
| 19 | 关键字（static/final/this/super） | ⬜ 待复习 | 📌 重要 |
| 20 | 动态代理 | ⬜ 待复习 | 📌 重要 |
| 21 | JDK8 新特性 | ⬜ 待复习 | 📌 重要 |
| 22 | I/O 流 / BIO-NIO-AIO | ⬜ 待复习 | 🔵 了解 |
| 23 | SPI 机制 | ⬜ 待复习 | 🔵 了解 |
| 24 | 语法糖 | ⬜ 待复习 | 🔵 了解 |
| 25 | BigDecimal 精度 | ⬜ 待复习 | 🔵 了解 |

---

## 三、今日复习策略（4/30）

### 上午：核心必补 🔥（约 3h）
1. **Runnable vs Callable** → 衔接多线程模块 [[Runnable vs Callable]]
2. **反射机制** → 框架基石 [[反射]]
3. **异常处理全体系** → 高频必问 [[异常处理]]
4. **包装类 + 缓存池** → 配合 == vs equals 理解 [[包装类]]

### 下午：重要补充 📌（约 2h）
5. **Object 类方法** → wait/notify/clone 等 [[Object类方法]]
6. **值传递 vs 引用传递** → 面试经典陷阱 [[值传递]]
7. **深拷贝 vs 浅拷贝**
8. **内部类（4 种）**
9. **注解**
10. **序列化**
11. **关键字深度** (static/final)
12. **动态代理**

### 晚间：快速过 + 串连 🔵（约 1h）
- JDK8 新特性（Lambda/Stream/Optional）
- I/O 流 / BIO-NIO-AIO 概念
- SPI 机制 / 语法糖 / BigDecimal
- 整体串连 + 话术梳理

---

## 四、关联模块

- [[../多线程/多线程总览|多线程与并发编程]] — 多线程 + 锁 + 线程池
- [[../Java集合/]] — 集合框架深度
- [[../JVM/]] — 类加载 + 内存模型 + GC
- [[../SSM框架/SSM总览|SSM框架]] — Spring IOC/AOP 涉及反射/动态代理

---

## 参考来源

- 飞书文档「java基础」(U5Azw8VKoi8bcHkj0vpcp6lFnyf)
- 飞书多维表格「Java八股复习计划」(ZyVJbwHEiaYUupsx7akca4BRnVg)
