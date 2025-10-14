---
tags: [八股, java基础, 多线程]
status: 已完成
---

# Runnable 和 Callable

> [!summary] 我的速记
> **一句话：R 无返无异常，C 有返有异常。**
> - Runnable：`void run()` — 不返回值、不抛 checked 异常
> - Callable：`V call()` — 有返回值（泛型）、可抛 checked 异常
> - Callable 不能直接传 Thread，要用 `FutureTask` 包装（FutureTask 同时实现了 Runnable 和 Future）
> - 实际开发用线程池：`executor.submit(callable)` 返回 `Future<V>`

## 1. 线程创建的 4 种方式概述

| 方式 | 说明 |
|------|------|
| 继承 `Thread` | 重写 `run()`，调用 `start()` 启动 |
| 实现 `Runnable` | 实现 `run()`，通过 `new Thread(runnable)` 启动 |
| 实现 `Callable` + `FutureTask` | 实现 `call()`，有返回值，可抛异常 |
| 线程池 | 通过 `Executors` 框架统一管理线程资源 |

## 2. Runnable vs Callable 核心对比

| 对比维度 | Runnable | Callable |
|----------|----------|----------|
| 所在包 | `java.lang` | `java.util.concurrent` |
| 方法签名 | `void run()` | `V call() throws Exception` |
| 返回值 | 无返回值 | 有返回值（泛型 `V`） |
| 异常处理 | 不能抛出 checked exception | 可以抛出 checked exception |
| 启动方式 | `new Thread(runnable).start()` | 需 `FutureTask` 包装后再传给 `Thread` |
| 提交线程池 | `executor.execute(runnable)` | `executor.submit(callable)` 返回 `Future<V>` |

## 3. Callable 使用方式

Callable 不能直接传入 `Thread` 的构造器，需要先用 `FutureTask` 包装：

```java
// 1. 定义 Callable 任务
Callable<Integer> callable = () -> {
    Thread.sleep(1000);
    return 42;
};

// 2. 用 FutureTask 包装
FutureTask<Integer> futureTask = new FutureTask<>(callable);

// 3. 通过 Thread 启动
new Thread(futureTask).start();

// 4. 获取返回值（会阻塞直到计算完成）
Integer result = futureTask.get();
System.out.println("结果: " + result); // 输出: 结果: 42
```

## 4. Future 和 FutureTask 的关系

```
Future (接口)        Runnable (接口)
    ↑                    ↑
    └──── RunnableFuture (接口) ────┘
                ↑
          FutureTask (实现类)
```

`FutureTask` 同时实现了 `RunnableFuture` 接口，而 `RunnableFuture` 继承了 `Runnable` 和 `Future`，所以 `FutureTask` **既可以作为任务被线程执行，又可以通过 Future 接口获取执行结果**。

核心方法：
- `get()` — 阻塞获取结果
- `get(timeout, unit)` — 超时获取结果
- `cancel(boolean)` — 取消任务
- `isDone()` — 判断任务是否完成

## 5. 为什么需要 Callable？

> 当多线程计算需要**返回结果**时，Runnable 无法满足需求，Callable 填补了这个空白。

典型场景：
- 多个线程并行计算，主线程汇总结果
- 异步任务执行后需要获取返回值
- 需要捕获子线程抛出的异常并统一处理

## 6. 简单示例代码

### Runnable 示例

```java
Runnable runnable = new Runnable() {
    @Override
    public void run() {
        // 模拟计算
        int sum = 0;
        for (int i = 0; i < 100; i++) {
            sum += i;
        }
        // ❌ 无法直接返回 sum，只能存到共享变量或打印
        System.out.println("Runnable 计算结果: " + sum);
    }
};
new Thread(runnable).start();
```

### Callable 示例

```java
Callable<Integer> callable = () -> {
    int sum = 0;
    for (int i = 0; i < 100; i++) {
        sum += i;
    }
    return sum; // ✅ 直接返回结果
};

FutureTask<Integer> task = new FutureTask<>(callable);
new Thread(task).start();
System.out.println("Callable 计算结果: " + task.get());
```

### 线程池 + Callable（更常用）

```java
ExecutorService executor = Executors.newFixedThreadPool(3);
Future<Integer> future = executor.submit(() -> {
    // 耗时计算
    return 42;
});
Integer result = future.get(); // 阻塞等待结果
executor.shutdown();
```

## 原理深挖

### FutureTask 的适配器模式

FutureTask 实现了 `RunnableFuture<V>`，而 `RunnableFuture<V>` 同时 extends `Runnable` 和 `Future<V>`。这就是适配器模式的经典应用：Callable 本身不能被 Thread 接受 → FutureTask 把它包装成 Runnable → Thread 就能用了。执行完后通过 Future 接口拿到结果——**一个类同时承担了「任务」和「结果容器」两个角色**。

### get() 的阻塞机制

```java
// FutureTask.get() 的内部简化逻辑
public V get() {
    int s = state;
    if (s <= COMPLETING)           // 任务还没完成
        s = awaitDone(false, 0L);  // 当前线程进入等待队列，park 阻塞
    return report(s);              // 完成，返回结果
}
// 底层是 LockSupport.park()——和 synchronized/wait 不同，不需要在同步块中
```

---

## 面试回答框架

> [!info] 问：Runnable 和 Callable 的区别？

**① 一句话定位：** Runnable 无返回值不抛异常，Callable 有泛型返回值可抛异常。Callable 需要用 FutureTask 包装后才能传入 Thread。

**② 核心展开：** Runnable 的 run() 返回 void，Callable 的 call() 返回泛型 V 且可抛 checked 异常。FutureTask 是关键适配器——它同时实现了 Runnable 和 Future 接口，执行完成后通过 get() 获取结果（阻塞直到完成）。

**③ 实际使用：** 日常开发中基本用线程池——`executor.submit(callable)` 返回 `Future<V>`，无需手动 FutureTask 包装。需要超时控制时用 `future.get(timeout, unit)`，捕获 `TimeoutException`。

---

## 跨题串联

| 引到 | 衔接 |
|------|------|
| [[../03-核心机制/异常处理]] | 「Runnable 不能抛 checked 异常——这就是 Callable 设计的核心动机之一。线程池的 submit 返回 Future，get 时会把执行中的异常包装成 ExecutionException 抛出」 |
| [[../../多线程/05-线程池/线程池详解]] | 「线程池的 execute() 只接受 Runnable，submit() 接受 Callable 返回 Future——这是两种提交方式的本质区别」 |

---

## 记忆口诀

> **R 无返无异常，C 有返有异常；F T 包装一下，线程池里 submit它。**
