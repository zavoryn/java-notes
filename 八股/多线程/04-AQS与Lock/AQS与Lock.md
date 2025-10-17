---
tags:
  - 八股
  - 多线程
  - AQS
  - Lock
status: 复习中
created: 2026-05-02
---

# AQS 与 Lock 体系

> [!summary] 我的速记
> **AQS = 一个 int state（状态） + 一个 CLH 双向队列（排队） + 模板方法。** 整个 Lock 体系的基石。
> **获取锁：** CAS 抢 state（0→1），抢不到就入队 park。**释放锁：** state 改回 0，unpark 队头线程。
> **ReentrantLock 重入靠什么？** state 累加——同一个线程再次获取时 state++，释放时 state--，直到 0 才释放。

---

## 一、AQS（AbstractQueuedSynchronizer）——Lock 体系的基石

### 1.1 核心三要素

```
AQS = state (volatile int) + CLH 队列 + 模板方法模式
       ↑                     ↑
   锁的状态/重入次数      等待线程的排队队列
```

```java
// AQS 核心字段
public abstract class AbstractQueuedSynchronizer {
    private volatile int state;                    // 同步状态
    private transient volatile Node head;          // CLH 队列头
    private transient volatile Node tail;          // CLH 队列尾

    // 独占模式
    protected boolean tryAcquire(int arg);         // 子类实现：尝试获取锁
    protected boolean tryRelease(int arg);         // 子类实现：尝试释放锁

    // 共享模式
    protected int tryAcquireShared(int arg);       // 子类实现：共享获取
    protected boolean tryReleaseShared(int arg);   // 子类实现：共享释放
}
```

> 🧠 **模板方法模式：** AQS 定义了获取锁的骨架（`acquire()` 方法），但"能不能获取"这个判断留给子类（`tryAcquire()`）。ReentrantLock、Semaphore、CountDownLatch 都只要实现自己的 `tryAcquire/tryRelease` 即可。

---

### 1.2 CLH 队列——等待线程怎么排队？

```
CLH 队列（变种——双向链表 + park/unpark）

  head (哑节点)           Node 1              Node 2              tail
┌────────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│  thread: null  │◄──│  thread: T1 │◄──│  thread: T2 │◄──│  thread: T3 │
│  waitStatus: 0 │──►│  ws: SIGNAL │──►│  ws: SIGNAL │──►│  ws: 0      │
└────────────────┘   └─────────────┘   └─────────────┘   └─────────────┘

  head 是已获取到锁的线程（出队后变哑节点）
  新线程入队时 CAS 更新 tail
  前一个节点的 ws=SIGNAL 时，park 当前线程
  锁释放时 unpark head.next
```

**Node 的 waitStatus（关键状态）：**

| ws | 值 | 含义 |
|----|------|------|
| CANCELLED | 1 | 节点取消（超时/中断） |
| SIGNAL | -1 | **后续节点需要被唤醒**——prev 释放锁时 unpark next |
| CONDITION | -2 | 节点在 Condition 等待队列中 |
| PROPAGATE | -3 | 共享模式下传播唤醒 |
| 0 | 0 | 初始状态 |

> 🧠 **SIGNAL 是核心：** 前一个节点退出时，检查自己的 ws 是否为 SIGNAL → 是则 unpark 后继节点。

---

### 1.3 获取锁的完整流程

```java
// AQS.acquire() — 独占获取锁的骨架
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&                    // ① 子类尝试一次获取
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))  // ② 失败则入队 + park
        selfInterrupt();
}

// 入队 + park 循环
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {                             // 自旋
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {// ③ 自己是老二 → 再试一次
                setHead(node);                 // 成功 → 变成新 head
                p.next = null;
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&  // ④ 前面的人 ws=SIGNAL?
                parkAndCheckInterrupt())                  // ⑤ 是 → park 自己
                interrupted = true;
        }
    } finally {
        if (failed) cancelAcquire(node);       // 出队
    }
}
```

> 🧠 **核心流程：tryAcquire 快速尝试 → 失败则入队 → park 自己 → 被前驱唤醒后再抢。**

---

## 二、ReentrantLock——可重入锁

### 2.1 为什么要可重入？

```java
public class Demo {
    Lock lock = new ReentrantLock();

    public void outer() {
        lock.lock();        // 获取锁，state: 0→1
        inner();            // 调用另一个加锁方法
        lock.unlock();      // state: 1→0
    }

    public void inner() {
        lock.lock();        // 再次获取锁 → 如果是不可重入锁，这里就死锁了！
                            // ReentrantLock: state: 1→2 ✅
        // ...
        lock.unlock();      // state: 2→1
    }
}
```

> 🧠 **重入就是："同一个线程可以多次获取同一把锁"。底层原理：state 累加，记录重入次数。释放时 state--，state==0 时才真正释放。**

### 2.2 公平锁 vs 非公平锁

```java
// 非公平锁（默认）——性能好，可能饥饿
Lock unfairLock = new ReentrantLock();  // 或 new ReentrantLock(false)

// 公平锁——FIFO，无饥饿但性能差
Lock fairLock = new ReentrantLock(true);
```

| | 非公平锁 | 公平锁 |
|---|---|---|
| tryAcquire | 直接 CAS 抢 state | 先检查队列里有没有人在等 |
| 性能 | ✅ 高——减少线程切换 | ❌ 低——每次都走队列 |
| 公平性 | 可能饥饿（新来的线程抢到锁） | ✅ 先进先出 |
| 默认 | ✅ 默认 | — |

**为什么非公平性能好？** 刚释放锁的线程可能还在 CPU 核上活跃，让它直接拿到锁继续执行——比唤醒队列里的线程（线程切换开销）更快。

---

## 三、Condition——多条件队列

```java
Lock lock = new ReentrantLock();
Condition notFull = lock.newCondition();   // 条件 1：缓冲区没满
Condition notEmpty = lock.newCondition();  // 条件 2：缓冲区不空

// 生产者
lock.lock();
try {
    while (buffer.isFull()) {
        notFull.await();       // 满了 → 释放锁，等待 notFull 条件
    }
    buffer.add(item);
    notEmpty.signal();          // 通知消费者：有东西了
} finally {
    lock.unlock();
}

// 消费者
lock.lock();
try {
    while (buffer.isEmpty()) {
        notEmpty.await();       // 空了 → 释放锁，等待 notEmpty 条件
    }
    buffer.take();
    notFull.signal();           // 通知生产者：有空间了
} finally {
    lock.unlock();
}
```

> 🧠 **synchronized 的 wait/notify 只有一个条件队列——notify 分不清通知的是生产者还是消费者。Condition 可以创建多个条件队列，精准唤醒。**

**Condition 内部原理：**

```
AQS 同步队列（获取锁的队列）
  head ← T1 ← T2 ← T3 ← tail

Condition 条件队列（等待条件的队列，单向链表）
  firstWaiter → Node(c1) → Node(c2) → Node(c3)

signal()：把条件队列的头节点移到 AQS 同步队列尾部
await()：当前线程加入条件队列，释放锁，park 自己
```

---

## 四、ReentrantReadWriteLock——读写锁

```
读锁 = 共享锁（多个线程可以同时持有读锁）
写锁 = 独占锁（同一时间只能一个线程写）

规则：
  - 有线程在读 → 新来的读可以进（共享），新来的写阻塞
  - 有线程在写 → 新来的读和新来的写都阻塞
  - 写锁可以降级为读锁（持有写锁时再获取读锁，然后释放写锁）
  - 读锁不能升级为写锁！
```

```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();

// 读操作（可并发）
rwLock.readLock().lock();
try {
    return cache.get(key);
} finally {
    rwLock.readLock().unlock();
}

// 写操作（独占）
rwLock.writeLock().lock();
try {
    cache.put(key, value);
} finally {
    rwLock.writeLock().unlock();
}
```

**state 怎么区分读锁和写锁？**

```
32 位 state 被拆成两半：
  高 16 位 = 读锁数量（共享计数）
  低 16 位 = 写锁重入次数（独占计数）

写锁获取：CAS 尝试将低 16 位从 0 → 1
读锁获取：CAS 将高 16 位 +1，成功后还需要 ThreadLocal 记录各自线程的重入次数
```

---

## 五、并发工具类——都是 AQS 的子类

| 工具类 | 基于 AQS | state 的含义 | 核心方法 |
|--------|----------|-------------|----------|
| **CountDownLatch** | 共享模式 | state = 计数（n→0） | `countDown()` state--；`await()` 等 state==0 |
| **Semaphore** | 共享模式 | state = 许可证数量 | `acquire()` state--；`release()` state++ |
| **CyclicBarrier** | ReentrantLock + Condition | 不是 AQS 子类！内部用 Lock + Condition | `await()` 等人齐，可复用 |

### CountDownLatch vs CyclicBarrier（面试高频对比）

| | CountDownLatch | CyclicBarrier |
|---|---|---|
| 比喻 | 倒计时门闩（一次性） | 循环栅栏（可复用） |
| 工作方式 | 一个线程等 N 个线程完成 | N 个线程相互等待，人齐了一起出发 |
| 复用 | ❌ 计数归零后不能重置 | ✅ `reset()` 可复用 |
| 回调 | 无 | `Runnable` 参数——人齐后先执行一段代码 |
| 底层 | AQS 共享模式 | ReentrantLock + Condition |

```java
// CountDownLatch
CountDownLatch latch = new CountDownLatch(3);
executor.submit(() -> { doWork(); latch.countDown(); });  // ×3
latch.await();  // 等 3 个线程都完成

// CyclicBarrier
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("人齐了，出发！"));
executor.submit(() -> { doPart(); barrier.await(); });  // ×3
// 3 个线程都到达后 → 执行回调 → 一起继续
```

---

## 六、面试满分话术

> **面试官：** 讲讲 AQS 和 ReentrantLock 的原理？

**你的回答：**

> **第一层（AQS 本质）：** AQS 是 Java 锁体系的基石——一个 int state + 一个 CLH 双向队列 + 模板方法模式。所有锁工具类（ReentrantLock、CountDownLatch、Semaphore）都是继承 AQS 实现的，只需要实现 `tryAcquire/tryRelease` 这几个模板方法就行。
>
> **第二层（ReentrantLock 流程）：** 获取锁时先 CAS 尝试将 state 从 0 改成 1——成功就直接拿到锁，失败就创建 Node 节点加入 CLH 队列尾部，然后 park 挂起自己。前驱节点释放锁时会检查自己的 waitStatus 是否为 SIGNAL——是则 unpark 后继节点。可重入通过在 state 上累加实现——同一个线程再次获取锁时 state++，释放时 state--，直到 0 才真正释放。
>
> **第三层（公平与非公平）：** 区别只在 `tryAcquire` 这一步——非公平锁直接 CAS 抢 state，公平锁先检查队列里是否有人在排队。非公平锁性能更好（刚释放锁的线程可能还在 CPU 上直接抢走），但可能造成饥饿。
>
> **第四层（Condition 和读写锁）：** Condition 解决了 synchronized 只有一个条件队列的问题——可以创建多个 Condition，精准唤醒（生产者只唤醒消费者，反之亦然）。读写锁里 state 拆成高 16 位（读锁共享计数）和低 16 位（写锁重入），读锁共享、写锁独占，支持锁降级但不支持升级。

> **追问：CountDownLatch 和 CyclicBarrier 的区别？**
>
> CountDownLatch 是一个线程等 N 个线程完成（一等多），CyclicBarrier 是 N 个线程互相等待（多等多）。CountDownLatch 基于 AQS 共享模式实现，计数归零后不能重置；CyclicBarrier 内部用 ReentrantLock + Condition 实现，可以 reset 重复使用。

---

## 常见面试问题清单

| #   | 问题                                     | 回答要点                                                         |
| --- | -------------------------------------- | ------------------------------------------------------------ |
| Q1  | **AQS 是什么？核心三要素？**                     | int state + CLH 双向队列 + 模板方法模式。Lock 体系的基石                     |
| Q2  | **ReentrantLock 锁获取流程？**               | CAS 抢 state(0→1)→成功则拿到锁→失败则入 CLH 队列尾部→park 挂起                |
| Q3  | **可重入怎么实现？**                           | state 累加。同一线程重入时 state++，释放时 state--，直到 0 才真正释放（unpark 后继）   |
| Q4  | **公平锁 vs 非公平锁区别？**                     | 公平锁先检查队列有没有人等；非公平直接 CAS 抢。非公平性能好但可能饥饿                        |
| Q5  | **Condition 和 wait/notify 区别？**        | Condition 支持多条件队列精准唤醒；wait/notify 只有一个队列，notify 随机唤醒         |
| Q6  | **CountDownLatch 和 CyclicBarrier 区别？** | Latch：一等多，不可复用，AQS 共享；Barrier：互相等，可 reset，内部用 Lock+Condition |

---

## 七、跨题串联

| 引到 | 衔接 |
|------|------|
| [[../02-synchronized锁升级/synchronized锁升级\|synchronized 锁升级]] | 「synchronized 基于 Monitor，ReentrantLock 基于 AQS——Lock 提供了公平锁、超时尝试、多条件等 Monitor 没有的高级功能」 |
| [[../03-CAS与Atomic/CAS与Atomic\|CAS 与 Atomic]] | 「AQS 的 state 修改全靠 CAS——整个 Lock 体系建立在一个 CAS 变量之上」 |
| [[../01-JMM与volatile/JMM与volatile\|JMM 与 volatile]] | 「AQS 的 state 是 volatile——保证多线程的可见性，CAS 保证原子修改」 |
| [[../05-线程池/线程池详解\|线程池]] | 「线程池的 shutdown 原理就是用 AQS 的 state 做状态控制」 |

---

## 八、记忆口诀

```
AQS 三要素，state 队列模板法。
队列是 CLH，双向链表 park unPark。
重入靠累加，释放减到零。
公平先排队，非公直接抢。
Condition 多条件，精准唤醒不广播。
CountDown 一等多，CyclicBarrier 互相等。
```

---

## 九、播面补充 🎧

> 以下内容抓取自**播面**（bomianfm.com），补充本笔记中未深入展开的知识点。

| 笔记 | 补充内容 |
|------|---------|
| [[播面-AQS独占与共享模式]] | 独占vs共享模式深度对比：唤醒传播机制、tryAcquireShared返回值语义、Node标记、读写锁混合使用 |

> [!note] 播面深化的知识点（本笔记涉及但未展开）
> - **唤醒机制**：独占「点对点」vs 共享「传播式连锁唤醒」
> - **tryAcquireShared 返回值**：int 类型（<0 失败 / =0 成功无余 / >0 成功有余）是驱动传播的关键
> - **Node 的 EXCLUSIVE/SHARED 标记**：队列中可混合存在
> - **ReentrantReadWriteLock 混合模式**：state 高低 16 位拆分，读写锁共享+独占协作
