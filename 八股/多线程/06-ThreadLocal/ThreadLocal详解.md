---
tags:
  - 八股
  - 多线程
  - ThreadLocal
status: 复习中
created: 2026-05-02
---

# ThreadLocal 原理与内存泄漏

> [!summary] 我的速记
> **ThreadLocal = 每个线程的"私人储物柜"。** 存在 Thread.threadLocals 里（ThreadLocalMap），key 是 ThreadLocal 的弱引用，value 是你要存的对象。
> **内存泄漏根因：** key 是弱引用会被 GC，value 是强引用 → Entry 的 value 永远不会被自动清理 → 线程不死内存不释放。
> **解决方案：用完了必须 `remove()`！** 线程池场景尤其危险。

---

## 一、ThreadLocal 是什么？——线程级别的隔离

```java
// 每个线程有自己的 ThreadLocal 副本，互不干扰
ThreadLocal<String> threadLocal = new ThreadLocal<>();

new Thread(() -> {
    threadLocal.set("线程A的数据");
    threadLocal.get();  // "线程A的数据"
}).start();

new Thread(() -> {
    threadLocal.set("线程B的数据");
    threadLocal.get();  // "线程B的数据"
}).start();
```

**实现机制：** 数据不是存在 ThreadLocal 对象里，是存在 Thread 对象的 `threadLocals` 字段里。

> [!warning] 常见误区
> 很多人误以为 `ThreadLocal` 内部有一个 `Map<Thread, T>` 来存数据——**这是错的！** 如果那样，ThreadLocal 持有 Thread 的强引用，会导致线程无法被 GC。真正的设计是反过来的：**Thread 持有 ThreadLocalMap**，ThreadLocal 只是 Key。

```
ThreadLocal 对象
     │
     │ 每个 Thread 内部有一个 ThreadLocalMap
     ▼
┌─────────────────────────────────┐
│ Thread A                        │
│   threadLocals = ThreadLocalMap │
│     Entry[0] → Key(弱引用)→TL1  │
│                Value → "A的数据" │
│     Entry[1] → Key(弱引用)→TL2  │
│                Value → 对象X     │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│ Thread B                        │
│   threadLocals = ThreadLocalMap │
│     Entry[0] → Key(弱引用)→TL1  │
│                Value → "B的数据" │
└─────────────────────────────────┘
```

> 🧠 **核心洞察：ThreadLocal 只是个"钥匙"，真正的数据存在 Thread 里面。同一个 ThreadLocal 对象在不同线程中对应不同的值——因为每个线程有自己的 Map。**

**面试加分比喻——酒店房卡 🏨：**
> 想象你去酒店，前台有一大堆房卡（`ThreadLocal` 对象）。你用身份证（`Thread.currentThread()`）领一张专属房卡，只能打开自己的房间（线程私有数据）。其他客人也用各自的房卡开各自的房间。房卡本身是共享的，但房间是私有的。

---

## 二、源码关键结构

```java
// Thread 类里的字段
public class Thread {
    ThreadLocal.ThreadLocalMap threadLocals;  // 存 ThreadLocal 的值
    ThreadLocal.ThreadLocalMap inheritableThreadLocals;  // 可继承的
}

// ThreadLocalMap 的 Entry
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;  // 你存的数据（强引用！）
    Entry(ThreadLocal<?> k, Object v) {
        super(k);  // key 是弱引用
        value = v;
    }
}
```

> 🧠 **关键设计：Entry 的 key（ThreadLocal 引用）是弱引用，但 value 是强引用。这个不对称设计是内存泄漏的根本原因。**

---

## 三、set/get 流程

```java
// ThreadLocal.set()
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);          // 拿到当前线程的 ThreadLocalMap
    if (map != null)
        map.set(this, value);                // key = 当前 ThreadLocal 对象引用
    else
        createMap(t, value);                 // 第一次 → 创建 Map
}

// ThreadLocal.get()
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);  // 用 this 查 key
        if (e != null) return (T) e.value;
    }
    return setInitialValue();  // 没找到 → 调 initialValue() 返回默认值
}
```

**ThreadLocalMap 的哈希冲突怎么解决？**

ThreadLocalMap 不是 JDK HashMap（链表法），而是用**开放地址法（线性探测）**：

```
hash = threadLocalHashCode & (len - 1)
冲突时 → hash+1 → hash+2 → ... → 找到第一个空位

为什么不用链表？ThreadLocalMap 的 key 是弱引用，Entry 数量通常很少，
开放地址法实现更简单，不需要链表节点额外开销。
```

---

## 四、内存泄漏——ThreadLocal 最大的坑

### 4.1 泄漏原理

```
ThreadLocal 对象（强引用）→ Stack 上的局部变量
     ↑
     │ 弱引用（GC 时会断开）
     │
  Entry {
      key   = WeakReference → ThreadLocal  ← 这个引用是弱的
      value = Object         ← 这个引用是强的！不会被 GC！
  }
```

```
泄漏路径：
1. 方法执行完 → ThreadLocal 引用出栈 → 没有强引用指向 ThreadLocal 对象
2. GC 发现 ThreadLocal 只有弱引用 → 回收 ThreadLocal 对象
3. Entry 的 key 变成 null，但 value 还在（强引用）！
4. 只要线程还活着（Thread.threadLocals 还引用着这个 Entry）→ value 永远不会被回收
5. 线程池中线程长期存活 → value 一直堆积 → 内存泄漏！
```

> 🧠 **一句话：key 是弱引用被 GC 了，但 value 是强引用，Entry 还挂在 Thread 上——线程活着就永远漏。**

### 4.2 JDK 的缓解措施

ThreadLocalMap 的 `get/set/remove` 方法会顺便清理 key 为 null 的 Entry：

```java
// get 时如果发现 key 为 null
//   → expungeStaleEntry() 清理这个 Entry
// remove 时
//   → 显式断开 value 引用 + 清理相邻的过期 Entry
```

但这只是**被动清理**——如果你不再调用 get/set/remove，过期 Entry 就永远留在那里。

### 4.3 正确的使用姿势

```java
// ❌ 错误：没有 remove，线程池场景必漏
ThreadLocal<User> local = new ThreadLocal<>();
try {
    local.set(user);
    // 业务逻辑...
    // 忘记 remove()！
} finally {
    // 什么都没做
}

// ✅ 正确：finally 里必须 remove
ThreadLocal<User> local = new ThreadLocal<>();
try {
    local.set(user);
    // 业务逻辑...
} finally {
    local.remove();  // 必须！
}
```

---

## 五、四种引用类型速记（面试加分）

| 引用类型 | GC 行为 | 典型应用 |
|----------|--------|----------|
| **强引用** | 永远不会被 GC（OOM 也不回收） | `Object obj = new Object()` |
| **软引用** | 内存不足时回收 | 图片缓存、网页缓存 |
| **弱引用** | 下一次 GC 就回收 | **ThreadLocalMap 的 key**、WeakHashMap |
| **虚引用** | 随时被回收，仅用于跟踪回收事件 | NIO DirectByteBuffer 的 Cleaner |

> 🧠 **ThreadLocal 为什么用弱引用而不是强引用？** 如果 key 是强引用，即使业务代码不再持有 ThreadLocal 对象，它也会因为 Entry 持有强引用而无法被 GC——泄漏更严重。弱引用至少让 ThreadLocal 对象本身可以被回收。

---

## 六、InheritableThreadLocal——父子线程传递

```java
InheritableThreadLocal<String> context = new InheritableThreadLocal<>();

context.set("父线程的上下文");

new Thread(() -> {
    System.out.println(context.get());  // "父线程的上下文" ← 自动继承！
}).start();
```

**原理：** 创建子线程时，Thread 的 `init()` 方法会检查父线程的 `inheritableThreadLocals` → 如果非空 → 拷贝一份给子线程。

> 🧠 **注意：** 是**拷贝**不是共享——子线程修改不影响父线程，父线程后续修改也不影响已创建的子线程。

**线程池场景的问题：** 线程池的线程是复用的 → 只在第一次创建时继承 → 后续任务不会自动获取最新的父线程值。解决方案：使用阿里的 `TransmittableThreadLocal`（TTL）。

---

## 七、实际应用场景

| 场景 | 示例 |
|------|------|
| **数据库连接** | Spring 的 `TransactionSynchronizationManager` 用 ThreadLocal 存当前事务的 Connection |
| **Web 请求上下文** | 拦截器把用户信息存入 ThreadLocal → 后续业务代码随处可取 |
| **链路追踪** | `MDC.put("traceId", xxx)` 底层是 ThreadLocal |
| **日期格式化** | `SimpleDateFormat` 不是线程安全 → 每个线程一个实例 |

**SimpleDateFormat 的典型用法：**
```java
// ❌ 错误：多线程共享一个 SimpleDateFormat → 格式化结果错乱
private static final SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");

// ✅ 方案1：每次 new（开销大）
// ✅ 方案2：synchronized 加锁（并发性能差）
// ✅ 方案3：ThreadLocal — 每个线程一个实例，无锁且复用
private static final ThreadLocal<SimpleDateFormat> dateFormatLocal =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

public String format(Date date) {
    return dateFormatLocal.get().format(date);  // 每个线程用自己的 SDF
}
```

> 🧠 **SimpleDateFormat 是 ThreadLocal 最经典的用例——非线程安全的工具类，通过 ThreadLocal 实现"每个线程一份"，既不用锁也不反复创建。**
| **分页参数** | MyBatis 的 PageHelper 用 ThreadLocal 传分页参数 |

---

## 八、面试满分话术

> **面试官：** ThreadLocal 原理？为什么会有内存泄漏？

**你的回答：**

> **第一层（原理）：** ThreadLocal 是线程级别的数据隔离。每个 Thread 内部有一个 ThreadLocalMap，ThreadLocal 对象本身只是 key——真正的数据存在各自线程的 Map 里。set 时拿到当前线程的 Map，以 ThreadLocal 为 key 存入 value；get 同理。
>
> **第二层（内存泄漏）：** ThreadLocalMap 的 Entry 设计很特殊——key 是弱引用，value 是强引用。当 ThreadLocal 对象不再被业务代码引用时，GC 会回收 ThreadLocal 对象（弱引用），导致 Entry 的 key 变成 null。但 value 是强引用，只要线程还活着，value 就永远不会被回收——这就是内存泄漏。
>
> **第三层（为什么这么设计）：** 如果 key 是强引用，ThreadLocal 对象就永远不会被 GC——泄漏更严重。弱引用至少让 ThreadLocal 对象本身能回收。JDK 在 get/set/remove 时也会顺便清理 key 为 null 的 Entry，但这是被动清理。
>
> **第四层（解决方案+踩坑）：** 使用 ThreadLocal 必须 `remove()`，放在 finally 块里。线程池场景尤其危险——线程长期存活，不 remove 会持续堆积。另外 InheritableThreadLocal 支持父子线程传递，但在线程池场景失效（线程复用）——需要用阿里的 TransmittableThreadLocal。

> **追问：ThreadLocalMap 为什么用开放地址法而不是链表？**
>
> 因为 ThreadLocalMap 的 key 是弱引用，Entry 数量通常很少，冲突频率低。开放地址法不需要链表节点的额外内存开销，实现也更简单。而且开放地址法在配合过期 Entry 清理（线性探测过程中顺手清理）时更方便。

---

## 常见面试问题清单

| # | 问题 | 回答要点 |
|---|------|----------|
| Q1 | **ThreadLocal 原理？** | 每个 Thread 内部有 ThreadLocalMap，ThreadLocal 是 key，数据存在各自线程的 Map 里 |
| Q2 | **内存泄漏怎么产生的？** | Entry key 弱引用被 GC → key 变 null；value 强引用 + 线程存活 → 永远不清 |
| Q3 | **怎么解决内存泄漏？** | `remove()` 必须调，放在 finally 块。线程池场景尤其关键 |
| Q4 | **ThreadLocalMap 怎么解决哈希冲突？** | 开放地址法（线性探测），不是链地址法。冲突时往后找空位 |
| Q5 | **为什么 Entry 用弱引用？** | 如果 key 是强引用，ThreadLocal 对象永远不会被 GC→泄漏更严重。弱引用至少让 ThreadLocal 可回收 |
| Q6 | **InheritableThreadLocal 和线程池的问题？** | 子线程创建时拷贝父线程值，但线程池的线程复用导致不更新。用阿里的 TransmittableThreadLocal |

---

## 九、跨题串联

| 引到 | 衔接 |
|------|------|
| [[../05-线程池/线程池详解\|线程池]] | 「线程池 + ThreadLocal = 内存泄漏炸弹！线程复用导致 ThreadLocalMap 中的过期 Entry 不会自动清理——线程不死，内存不释放」 |
| [[../../java基础/04-字符串与常用类/equals和hashCode\|equals 和 hashCode]] | 「ThreadLocalMap 的哈希冲突用开放地址法（线性探测），HashMap 用链地址法——两种截然不同的解决思路」 |
| [[../../java基础/02-面向对象/三大特性\|三大特性（强/软/弱/虚引用）]] | 「ThreadLocal 的 Entry 用了弱引用——强引用 永远不会 GC、软引用 OOM 前回收、弱引用下次 GC 回收、虚引用随时回收」 |
| [[../04-AQS与Lock/AQS与Lock\|AQS 与 Lock]] | 「ReentrantReadWriteLock 的读锁重入计数也是用 ThreadLocal 实现的——每个线程记录自己获取了几次读锁」 |

---

## 十、记忆口诀

```
ThreadLocal 线程隔离，数据存在 Thread 里。
Entry 的 key 是弱引用，value 强引用是祸根。
Key 被 GC 变 null，value 永远清不掉。
线程池里长期活，不用 remove 必泄漏。
解决就一条——finally 里面 remove 掉！
```

---

## 十一、播面交叉验证 🎧

> 来源：播面 [ID:15](https://www.bomianfm.com/web/question/15)。本笔记与播面覆盖率 **~95%**，以下为补充点（已直接合并到正文）：

| 补充项 | 位置 |
|--------|------|
| 🏨 酒店房卡比喻 | → 第一节末尾 |
| ⚠️ `Map<Thread,T>` 常见误区 | → 第一节「实现机制」下方 |
| 📝 SimpleDateFormat 完整代码示例 | → 第七节「日期格式化」行 |

> 播面 5 道关联追问（数据隔离原理、适用场景、ThreadLocalMap 关系、内存泄漏原因、remove 必要性）——本笔记 Q1~Q6 已全部覆盖 ✅
