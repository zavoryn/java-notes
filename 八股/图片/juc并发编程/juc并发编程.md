# JUC 并发编程

## 线程基础 ★★★★

1. 进程与线程的区别，线程的状态及转换
2. 并发的优势与问题（上下文切换、死锁、线程安全、资源竞争）
3. 线程的创建方式（继承 Thread、实现 Runnable、Callable + Future、线程池）
4. Thread 类的核心方法（start、run、sleep、wait、notify、join、yield 等）
5. 线程的中断机制（interrupt()、isInterrupted()、interrupted()）（概率低，选择性学习）

## 同步机制 ★★★★ 全部是重点！！！

synchronized，volatile，Lock

1. synchronized 的底层实现（对象头、Monitor、偏向锁/轻量级锁/重量级锁升级）
2. synchronized 修饰方法、代码块的区别，锁的对象是什么？
3. volatile 的作用（可见性、禁止指令重排序），不能保证原子性的原因
4. volatile 的底层实现（内存屏障）
5. synchronized 与 volatile 的区别
6. Lock 接口及实现类（ReentrantLock、ReentrantReadWriteLock）的使用
7. synchronized 与 ReentrantLock 的区别（公平锁、可中断、条件变量等）
8. Java 中的原子类（AtomicInteger、AtomicReference 等）及 CAS 原理，ABA 问题

## 线程池 ★★★★ 全部是重点！！！

ThreadPoolExecutor, Executors

1. 线程池的核心参数（核心线程数、最大线程数、空闲时间、工作队列等）
2. 线程池的工作流程（任务提交后的执行流程）
3. 常见的线程池类型（FixedThreadPool、CachedThreadPool、ScheduledThreadPool 等）及问题
4. 线程池的拒绝策略（AbortPolicy、CallerRunsPolicy 等）及使用场景
5. 如何合理配置线程池参数？（CPU 密集型、IO 密集型区别）

## 并发工具类 ★★★

1. CountDownLatch、CyclicBarrier、Semaphore 的原理与使用场景（这个有可能让手撕）
2. ConcurrentHashMap 的底层实现（JDK7 分段锁，JDK8 数组+链表+红黑树+Synchronized）
3. ThreadLocal 的原理、使用场景及内存泄漏问题（弱引用的作用）
4. CopyOnWriteArrayList、CopyOnWriteArraySet 的原理、优缺点及使用场景
5. BlockingQueue 接口及实现类（ArrayBlockingQueue、LinkedBlockingQueue 等）的原理

## 并发问题 ★★★

1. 死锁的产生条件、如何避免与排查（jstack 命令）
2. 线程安全的定义，如何保证线程安全（无状态、不可变、同步机制等）
3. 上下文切换的概念及优化方式
