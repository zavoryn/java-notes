# Java 八股文知识库

> 面试知识点索引，按模块分类。点击链接跳转到对应文档。

## 目录

### [Java 基础](java基础/java基础.md)
> 来源：小红书知识卡片

| 模块 | 重要度 | 关键词 |
|------|--------|--------|
| [集合](java基础/java基础.md) | ★★★★★ | HashMap, ArrayList, LinkedList, HashSet, ConcurrentHashMap |
| [字符串](java基础/java基础.md) | ★★★★ | String, StringBuilder, StringBuffer, equals |
| [泛型](java基础/java基础.md) | ★★★★ | 类型擦除, 通配符, 泛型类/接口/方法 |
| [面向对象](java基础/java基础.md) | ★★★★ | 封装/继承/多态, 重写与重载, 抽象类与接口, final, static |
| [异常](java基础/java基础.md) | ★★ | Exception, Error, try-catch-finally |
| [其他基础](java基础/java基础.md) | ★★ | 包装类, 枚举, 注解, Java新特性 |

---

### [JUC 并发编程](juc并发编程/juc并发编程.md)
> 来源：小红书知识卡片

| 模块 | 重要度 | 关键词 |
|------|--------|--------|
| [线程基础](juc并发编程/juc并发编程.md) | ★★★★ | 线程创建, 生命周期, 核心方法 |
| [同步机制](juc并发编程/juc并发编程.md) | ★★★★ 重点 | synchronized, volatile, Lock, CAS, AQS |
| [线程池](juc并发编程/juc并发编程.md) | ★★★★ 重点 | ThreadPoolExecutor, 拒绝策略 |
| [并发工具类](juc并发编程/juc并发编程.md) | ★★★ | CountDownLatch, ConcurrentHashMap, ThreadLocal |
| [并发问题](juc并发编程/juc并发编程.md) | ★★★ | 死锁, 线程安全, 上下文切换 |

---

### [JVM 虚拟机](jvm虚拟机/jvm虚拟机.md)
> 来源：小红书知识卡片

| 模块 | 重要度 | 关键词 |
|------|--------|--------|
| [垃圾回收（GC）](jvm虚拟机/jvm虚拟机.md) | ★★★★★ | GC区域, 可达性分析, 收集算法, CMS, G1, ZGC |
| [JVM内存模型](jvm虚拟机/jvm虚拟机.md) | ★★★★ | 运行时数据区, 堆内存分段, OOM |
| [类加载机制](jvm虚拟机/jvm虚拟机.md) | ★★★ | 类加载生命周期, 双亲委派 |
| [性能调优](jvm虚拟机/jvm虚拟机.md) | ★★★★ | jps, jstat, jmap, jstack |
| [执行引擎](jvm虚拟机/jvm虚拟机.md) | ★ | JIT, 字节码执行, 方法分派 |

---

### [Spring](spring/spring.md)
> 来源：小红书知识卡片

| 模块 | 重要度 | 关键词 |
|------|--------|--------|
| [Spring核心概念](spring/spring.md) | ★★★★★ | IOC/DI, AOP, Bean |
| [Spring IOC容器](spring/spring.md) | ★★★ | BeanFactory, ApplicationContext, 生命周期, 作用域 |
| [Spring AOP](spring/spring.md) | ★★★ | JDK动态代理, CGLIB, 通知类型 |
| [核心问题与优化](spring/spring.md) | ★★★★★ | 循环依赖三级缓存, @Autowired vs @Resource |

---

### [MySQL](mysql/mysql.md)
> 来源：小红书知识卡片

| 模块 | 关键词 |
|------|--------|
| [数据库基础](mysql/mysql.md) | 存储引擎, ACID, 隔离级别, MVCC |
| [索引](mysql/mysql.md) | B+树, 最左匹配, 索引失效, 覆盖索引 |
| [SQL 优化](mysql/mysql.md) | Explain, 查询优化, 分页优化 |
| [锁机制](mysql/mysql.md) | 表锁/行锁, 间隙锁, 死锁, 乐观锁/悲观锁 |
| [日志与恢复](mysql/mysql.md) | binlog, redo log, undo log, 两阶段提交 |
| [其他核心问题](mysql/mysql.md) | 主从复制, 分库分表, 慢查询 |

---

### [Redis](redis/redis.md)
> 来源：小红书知识卡片

| 模块 | 重要度 | 关键词 |
|------|--------|--------|
| [Redis基础](redis/redis.md) | ★★★★★ | 数据类型, 过期策略, 内存淘汰 |
| [持久化](redis/redis.md) | ★★★★★ | RDB, AOF, AOF重写 |
| [缓存问题](redis/redis.md) | ★★★★★ | 缓存穿透, 击穿, 雪崩, 数据一致性 |
| [高可用](redis/redis.md) | ★★★★★ | 主从复制, 哨兵, Cluster集群 |
| [其他核心问题](redis/redis.md) | ★★★★ | 事务, 分布式锁, Pipeline |

---

### [消息队列](消息队列/消息队列.md)
> 来源：小红书知识卡片

| 模块 | 重要度 | 关键词 |
|------|--------|--------|
| [MQ基础](消息队列/消息队列.md) | ★★★★★ | 异步, 解耦, 削峰, 产品对比与选型 |
| [MQ核心问题](消息队列/消息队列.md) | ★★★★★ 重中之重 | 可靠性, 重复消费, 顺序性, 消息积压, 分布式事务 |
| [产品特性](消息队列/消息队列.md) | — | RabbitMQ, RocketMQ, Kafka |

---

### [微服务](微服务/微服务.md)
> 来源：[飞书云文档](https://my.feishu.cn/wiki/FLS2wDVlmiJUepknxyRc4P3Enjd)

| 面试题 | 关键词 |
|--------|--------|
| [Spring Cloud 5大组件](微服务/微服务.md) | Eureka, Ribbon, Feign, Hystrix, Gateway |
| [服务注册与发现](微服务/微服务.md) | DiscoveryClient, 心跳, AP/CP |
| [Nacos vs Eureka](微服务/微服务.md) | 临时实例, 非临时实例, 消息推送 |
| [负载均衡实现](微服务/微服务.md) | Nginx(服务端LB), Ribbon/LoadBalancer(客户端LB) |
| [自定义负载均衡策略](微服务/微服务.md) | IRule, ReactorServiceInstanceLoadBalancer, 灰度发布 |
| [服务雪崩](微服务/微服务.md) | 服务降级, 服务熔断, Hystrix, Sentinel |
| [微服务监控](微服务/微服务.md) | SkyWalking |
| [限流方案](微服务/微服务.md) | 漏桶, 令牌桶, Sentinel, Redis+Lua |
| [CAP/BASE理论](微服务/微服务.md) | 一致性, 可用性, 分区容错性 |
| [分布式事务](微服务/微服务.md) | Seata AT模式, TCC, MQ |
| [接口幂等性](微服务/微服务.md) | Token机制, 唯一索引, 乐观锁, 状态机 |
| [XXL-JOB任务调度](微服务/微服务.md) | 分片广播, 故障转移, 幂等性 |

---

### [计算机网络](计算机网络/计算机网络.md)
> 来源：小红书知识卡片

| 模块 | 重要度 | 关键词 |
|------|--------|--------|
| [网络分层模型](计算机网络/计算机网络.md) | ★★★★ | OSI七层, TCP/IP四层 |
| [网络层](计算机网络/计算机网络.md) | ★★★ | IP, ICMP, IPv6 |
| [传输层](计算机网络/计算机网络.md) | ★★★★★ 重点 | TCP/UDP, 三次握手, 四次挥手, 滑动窗口, 拥塞控制 |
| [应用层](计算机网络/计算机网络.md) | ★★★★★ | HTTP/HTTPS, DNS, 状态码 |

---

### [操作系统](操作系统/操作系统.md)
> 来源：小红书知识卡片

1. 进程、线程和协程的区别
2. 进程间通信的方式
3. 进程有哪些状态
4. 进程调度算法
5. 死锁
6. 内存管理方式
7. 页面置换算法
8. 虚拟内存
9. 磁盘调度算法
10. 用户态和内核态
11. 常见I/O模型
12. select、poll、epoll的区别

---

### [设计模式](设计模式/设计模式.md)
> 来源：小红书知识卡片

- 设计原则：SOLID + 迪米特法则
- 重点模式：单例、代理（动态/静态）、工厂、建造者、策略

---

### [AI Agent 知识](agent/)
> 来源：小红书知识卡片

| 模块 | 文件 | 关键词 |
|------|------|--------|
| MCP 协议 | [MCP](agent/mcp/后端同学要掌握的大模型知识-MCP.md) | Host/Client/Server、JSON-RPC、Stdio/HTTP+SSE |
| RAG | [RAG面试题](agent/rag/后端要掌握的RAG面试题.md) | RAG vs SFT、Chunk策略、向量数据库选型、准确率优化 |
| ReAct | [ReAct及面试题](agent/react/ReAct及面试题.md) | TAO闭环、CoT vs Act-only、LangChain/LangGraph |
| Memory | [Memory及面试题](agent/memory/Memory及面试题.md) | 短期/长期记忆、写入/整理/检索管线、向量数据库 |
| Skill | [Skill](agent/skill/后端要掌握的AI Agent知识-Skill.md) | Skill定义、与MCP关系、when_to_use、失败兜底 |
| Multi-Agent | [Multi-Agent](agent/mutiagent/Multi-Agent多智能体系统全解析.md) | 编排者-工作者、评估者-优化器、A2A协议 |
| 场景设计题 | [万能回答框架](agent/场景题/后端场景设计题万能回答框架.md) | 5个根本矛盾、需求分析→问题拆解→方案设计→综合讲述 |

---

## 开源经历

> 开源贡献经历，按项目维度准备，关联到知识点模块。
> 详见 → [开源经历/开源经历.md](开源经历/开源经历.md)

| PR | 项目 | 关键词 |
|----|------|--------|
| PR 1 | Nacos Copilot 配置测试能力建设 | LLM 配置校验, 可达性检测 |
| PR 2 | Nacos 配置中心灰度能力优化 | labels 表达式, 灰度发布, 规则模型 |
| PR 3 | Nacos MCP Registry 能力补齐 | MCP Resource, 元数据持久化, 级联清理 |

---

## 面试项目

> 项目经历是面试核心，按项目维度准备，关联到知识点模块。
> 详见 → [面试项目/README.md](面试项目/README.md)

| 项目 | 技术栈 | 核心亮点 | 详见 |
|------|--------|----------|------|
| 知光/颐享 | Spring Boot 3 + React 18 + Redis + ES + Kafka | 三级缓存Feed流、Bitmap幂等计数、发件箱+Canal CDC、RAG问答 | [CLAUDE.md](面试项目/yixiang/CLAUDE.md) |
| 多平台同步 | Spring Boot 2.7 + WebClient | 策略+模板+工厂三件套、三渠道防腐层、Webhook异步回调 | [CLAUDE.md](面试项目/多平台同步/CLAUDE.md) |
| 电商数据分析 | Spring Cloud + ClickHouse + RocketMQ + Flink | 三层幂等、双流Join补偿、三级查询、按月分表 | [CLAUDE.md](面试项目/电商平台/CLAUDE.md) |

---

## 博客

> 技术博客文章，同步发布于掘金和 CSDN。
> 掘金主页：https://juejin.cn/user/1329749731578971
> CSDN 主页：https://blog.csdn.net/qq_62915969?type=blog

| 文章 | 关键词 | 关联知识点 |
|------|--------|-----------|
| [从操作系统理解 AI Agent：进程、系统调用与上下文窗口](博客/从操作系统理解AI-Agent：进程、系统调用与上下文窗口.md) | AI Agent, OS类比, Tool Use, RAG, Context Window | [操作系统](操作系统/操作系统.md) |

---

## 面经索引

> 面经按面试场次记录，每个问题关联到对应知识点模块。

| 面经 | 日期 | 核心考点 |
|------|------|----------|
| [字节 - 后端开发（三轮面）](面经/流年/字节/字节_后端开发_2025-02-10.md) | 2025-02 | Redis三级缓存、MySQL MVCC/索引/B+树 |
| [tt直播 - 后端开发（二面）](面经/流年/tt直播/tt直播_后端开发_2026-04.md) | 2026-04 | Feed深分页、Bitmap点赞、线程池 |
| [千问 - c端开发（一面）](面经/流年/千问/千问_c端_2026-04-08.md) | 2026-04 | Redis SDS/Lua/Bitmap/HotKey、Outbox发件箱 |
| [不知名厂 - 后端开发](面经/不知名厂/不知名厂_后端开发_2026-04.md) | 2026-04 | Spring Boot/AOP/JWT、Redis、MySQL索引、TCP、线程安全 |
| [腾讯 - 后端开发](面经/流年/腾讯/腾讯_后端开发_2026-04.md) | 2026-04 | 微服务事件驱动、Canal+Kafka、HotKey、JVM GC、ThreadLocal |

---

## 知识来源

| 来源 | 类型 | 覆盖模块 |
|------|------|----------|
| 小红书 - 程序员流年 | 图片知识卡片 | Java基础, JUC, JVM, Spring, MySQL, Redis, 消息队列, 设计模式, 计算机网络, 操作系统 |
| [飞书云文档](https://my.feishu.cn/wiki/FLS2wDVlmiJUepknxyRc4P3Enjd) | 飞书文档 | 微服务 |
| [Java八股复习计划](https://my.feishu.cn/base/ZyVJbwHEiaYUupsx7akca4BRnVg) | 飞书多维表格 | 全局复习计划 |

---

## 统计

| 模块 | 考点数 | 来源 |
|------|--------|------|
| Java 基础 | 6大类 | 小红书 |
| JUC 并发编程 | 5大类 | 小红书 |
| JVM 虚拟机 | 5大类 | 小红书 |
| Spring | 4大类 | 小红书 |
| MySQL | 6大类 | 小红书 |
| Redis | 5大类 | 小红书 |
| 消息队列 | 3大类 | 小红书 |
| 微服务 | 18道面试题 | 飞书文档 |
| 计算机网络 | 4大类 | 小红书 |
| 操作系统 | 12道面试题 | 小红书 |
| 设计模式 | 原则+5大模式 | 小红书 |
