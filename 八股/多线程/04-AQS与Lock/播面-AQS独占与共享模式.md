---
tags:
  - 八股
  - 多线程
  - AQS
  - 播面
source: https://www.bomianfm.com/web/question/1191
created: 2026-05-02
---

# AQS 独占模式与共享模式

> 来源：播面 | ID: 1191
> 关联：[[AQS与Lock|AQS 与 Lock 体系]]

---

## 面试版速记总结

AQS 支持两种同步模式：**独占模式（Exclusive）** 和 **共享模式（Shared）**。根本区别在于**能否有多个线程同时获取到同步资源**。

### 核心对比

- **独占模式**：同一时刻**只能有一个线程**获取资源。比喻：单人洗手间。
- **共享模式**：同一时刻**可以有多个线程**同时获取资源（只要资源数量允许）。比喻：有 N 个座位的阅览室。

### 唤醒机制差异（最关键）

- **独占模式**：释放时仅唤醒**紧接着的下一个节点**（点对点）。
- **共享模式**：获取或释放时可**传播唤醒**后续共享节点，形成连锁反应直至资源耗尽。

### 需重写的方法

| 模式 | 方法 | 返回值 |
|:---|:---|:---|
| 独占 | `tryAcquire(int arg)` | `boolean`：true=成功 |
| 独占 | `tryRelease(int arg)` | `boolean`：true=释放成功 |
| 共享 | `tryAcquireShared(int arg)` | `int`：<0失败，=0成功无剩余，>0成功有剩余 |
| 共享 | `tryReleaseShared(int arg)` | `boolean`：true=可唤醒后续 |

### 典型应用

- 独占：**ReentrantLock**
- 共享：**Semaphore**、**CountDownLatch**、**ReentrantReadWriteLock 读锁**

### 可混合使用

ReentrantReadWriteLock 中：写锁独占 + 读锁共享，state 高 16 位存读锁计数，低 16 位存写锁计数。

---

## 思维导图

```
AQS 核心 🔥 (考察锁与同步器底层实现)
├── 同步状态: volatile int state
├── 等待队列: FIFO 双向 CLH 链表
├── 同步模式 🔥 (区分独占与共享差异)
│   ├── 独占模式: 单线程访问 (例: ReentrantLock)
│   └── 共享模式: 多线程并发访问 (例: Semaphore、CountDownLatch、ReadLock)
├── 唤醒机制 🔥 (高频差异点)
│   ├── 独占: 仅唤醒头结点后继 💀
│   └── 共享: 传播唤醒后续节点 💀
├── 需重写方法 🔥
│   ├── 独占: tryAcquire(arg) / tryRelease(arg)
│   └── 共享: tryAcquireShared(arg) 💀 (<0失败,=0满额,>0有余)
│            tryReleaseShared(arg)
├── 节点模式标记 💀: EXCLUSIVE / SHARED (队列可混合)
├── 可混合使用 🔥: ReentrantReadWriteLock
│   ├── state 高16位: 读锁计数
│   └── state 低16位: 写锁计数
└── 模板方法模式 🔥: 只实现 tryXxx，复用 acquire/release 逻辑
```

---

## 详细解答

在 Java 的并发编程中，**AQS（AbstractQueuedSynchronizer）** 是构建锁和同步器的核心框架。AQS 内部维护了一个 `volatile int state`（同步状态）和一个 FIFO 的双向链表（等待队列）。

AQS 支持两种同步模式：**独占模式（Exclusive）** 和 **共享模式（Shared）**。它们的根本区别在于**能否有多个线程同时获取到同步资源**。

---

### 1. 核心概念对比

**独占模式（Exclusive）：**
- **定义**：同一时刻，**只能有一个线程**获取到同步资源。其他尝试获取资源的线程都会被阻塞并加入到等待队列中。
- **比喻**：单人洗手间。一个人进去后门就锁上了，其他人必须在外面排队，等里面的人出来才能进去下一个。
- **典型应用**：`ReentrantLock`。

**共享模式（Shared）：**
- **定义**：同一时刻，**可以有多个线程**同时获取到同步资源（只要资源数量允许）。
- **比喻**：有 N 个座位的阅览室。只要还有空座位，多个人可以同时进去看书；如果没有空座位了，新来的人才需要排队。
- **典型应用**：`Semaphore`（信号量）、`CountDownLatch`（倒计时器）、`ReentrantReadWriteLock` 的读锁。

---

### 2. 唤醒机制的区别（最关键的底层差异）

在 AQS 的等待队列中，当持有锁的线程释放资源后，如何唤醒后续排队的线程，是这两种模式最大的不同：

**独占模式的唤醒（点对点）：**
- 当独占锁释放时，它只会唤醒等待队列中**紧接着的下一个节点**（即头节点的后继节点）。
- **流程**：线程 A 释放锁 → 唤醒线程 B → 线程 B 获取锁 → 结束。

**共享模式的唤醒（传播机制 Propagate）：**
- 当一个线程以共享模式成功获取资源后，如果资源还有剩余（`state > 0`），它**不仅自己会获取资源，还会主动唤醒下一个排队的共享节点**。下一个节点获取成功后，如果还有剩余资源，会继续唤醒下下个节点，形成**连锁反应（传播）**。
- 当共享锁释放时，也会触发唤醒后续节点的操作。
- **流程**：线程 A 获取共享锁 → 发现还有资源 → 唤醒排在后面的线程 B → 线程 B 获取锁 → 发现还有资源 → 唤醒线程 C... 直到资源耗尽。

---

### 3. 需要重写的方法不同

AQS 是基于模板方法模式设计的，自定义同步器时，需要根据使用哪种模式来重写不同的方法：

| 模式 | 需要重写的方法 | 返回值含义 |
| :--- | :--- | :--- |
| **独占模式** | `tryAcquire(int arg)` | `boolean`：`true` 表示获取成功，`false` 表示失败排队。 |
| | `tryRelease(int arg)` | `boolean`：`true` 表示释放成功并可以唤醒后续节点。 |
| **共享模式** | `tryAcquireShared(int arg)` | `int`：**< 0**：获取失败，需排队。**= 0**：获取成功，但没有剩余资源，无需唤醒后续。**> 0**：获取成功，且有剩余资源，需要唤醒后续排队节点。 |
| | `tryReleaseShared(int arg)`| `boolean`：`true` 表示释放成功且可以唤醒后续节点。 |

> ⚠️ **共享模式的关键差异**：`tryAcquireShared` 返回 `int` 而非 `boolean`，正数表示"获取成功且还有剩余"，这是驱动**传播唤醒**的信号。

---

### 4. 节点（Node）的标记不同

AQS 的等待队列中的节点（`Node`），在创建时会标记其等待的模式：

- **`Node.EXCLUSIVE`**：独占模式节点。表示该线程正在等待独占锁。
- **`Node.SHARED`**：共享模式节点。表示该线程正在等待共享锁。

> 一个 AQS 的队列中，是可以同时存在独占节点和共享节点的（例如 `ReentrantReadWriteLock` 的等待队列）。

---

### 5. 能否混合使用？

**可以混合使用。** AQS 允许在一个同步器中同时实现独占模式和共享模式。

**典型代表：`ReentrantReadWriteLock`（读写锁）**
- **写锁（WriteLock）** 使用了**独占模式**：写和写互斥、读和写互斥。调用 `acquire()` 和 `release()`。
- **读锁（ReadLock）** 使用了**共享模式**：读和读不互斥，多个线程可以同时读取。调用 `acquireShared()` 和 `releaseShared()`。
- 它巧妙地将 AQS 的一个 32 位的 `state` 变量拆分为两部分：高 16 位表示读锁（共享）的持有次数，低 16 位表示写锁（独占）的持有次数。

---

### 总结表

| 特性 | 独占模式 (Exclusive) | 共享模式 (Shared) |
| :--- | :--- | :--- |
| **资源访问** | 仅限单一线程 | 允许多个线程同时访问 |
| **state含义** | 通常 0 可用，1 被占用（重入递增） | 通常表示可用资源数量或屏障状态 |
| **获取方法** | `acquire()` / `tryAcquire()` | `acquireShared()` / `tryAcquireShared()` |
| **释放方法** | `release()` / `tryRelease()` | `releaseShared()` / `tryReleaseShared()` |
| **唤醒机制** | 仅唤醒队列中的下一个有效节点 | **传播机制（Propagate）**，不仅自己获取，还会唤醒后续共享节点 |
| **典型实现类** | `ReentrantLock` | `Semaphore`, `CountDownLatch` |

---

## 关联追问（5道）

### Q1: 独占模式与共享模式在 AQS 中的主要区别是什么？

- **独占模式**：同一时间仅一个线程可成功 acquire，释放时只唤醒一个后继节点。典型应用：ReentrantLock。
- **共享模式**：多个线程可同时 acquire 成功（只要资源计数允许），释放时可唤醒多个等待线程。典型应用：Semaphore、CountDownLatch、ReadLock。
- 理解 `tryAcquire/tryRelease`（独占）与 `tryAcquireShared/tryReleaseShared`（共享）的不同实现逻辑。

---

### Q2: 独占模式的唤醒机制和共享模式的唤醒机制有何不同？

- **独占模式**：唤醒时只选择单个等待线程（队列中第一个符合条件的线程），保证互斥执行。
- **共享模式**：唤醒时可连续释放多个等待线程，直到资源不足或达到上限。支持读多写少的并发优化，提高吞吐量。可能触发**连锁唤醒**（被唤醒线程继续释放资源并唤醒后续线程）。

---

### Q3: 自定义同步器时独占模式与共享模式分别要重写哪些方法？

- **独占模式**：必须重写 `tryAcquire(int arg)` 和 `tryRelease(int arg)`。可选重写 `isHeldExclusively()`。
- **共享模式**：必须重写 `tryAcquireShared(int arg)`（返回负值=失败，0=成功无剩余，正值=成功可继续共享）和 `tryReleaseShared(int arg)`。

---

### Q4: AQS 等待队列中的 Node 如何区分独占节点和共享节点？

Node 类中有 `SHARED` 常量表示共享节点，`EXCLUSIVE` 表示独占节点。Node 的 mode 在构造时确定，队列中独占与共享节点可混合存在，但唤醒策略不同：独占释放后仅唤醒后续一个节点，共享释放后会尝试连续唤醒多个共享节点，直到遇到独占节点为止。

---

### Q5: ReentrantReadWriteLock 怎样混合使用独占和共享模式？

- AQS 一个 int state 拆为：**高 16 位** = 读锁持有计数，**低 16 位** = 写锁重入次数。
- **写锁**走独占模式（`acquire/release`）：写锁持有时阻塞后续所有读写请求。
- **读锁**走共享模式（`acquireShared/releaseShared`）：多个线程可同时获取读锁，但存在写锁时阻塞。
- 支持**锁降级**（写→读），不支持锁升级（读→写）。
