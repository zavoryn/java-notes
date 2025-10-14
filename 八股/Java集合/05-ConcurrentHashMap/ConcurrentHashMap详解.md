---
tags:
  - 八股
  - Java集合
  - ConcurrentHashMap
status: 已完成
created: 2026-05-01
---

# ConcurrentHashMap 详解

> [!summary] 我的速记
> **JDK7 分段锁，JDK8 CAS + synchronized。**
> - JDK7：Segment 数组 + HashEntry，每个 Segment 继承 ReentrantLock，并发度 = Segment 数量（默认16）
> - JDK8：去掉 Segment，和 HashMap 同结构（数组+链表+红黑树），空桶 CAS 插入，非空桶 synchronized 锁头节点
> - sizeCtl：多义字段——初始容量 / 扩容阈值 / -1 初始化中 / -N 表示 N-1 个线程在扩容
> - 多线程协同扩容：每个线程认领一段区间迁移数据
> - 比 HashTable 好：HashTable 全表锁，CHM 粒度小得多

## 一、JDK7：分段锁（Segment）

### 1.1 数据结构

```text
ConcurrentHashMap (JDK7)
│
├── Segment[0]  ── ReentrantLock ── HashEntry[0] → HashEntry → HashEntry ...
├── Segment[1]  ── ReentrantLock ── HashEntry[0] → HashEntry → HashEntry ...
├── ...
└── Segment[15] ── ReentrantLock ── HashEntry[0] → HashEntry → HashEntry ...

每个 Segment = 一个小 HashMap（HashEntry 数组 + 链表）
默认 16 个 Segment，并发度 = Segment 数量
```

```java
// JDK7 核心结构
static final class Segment<K, V> extends ReentrantLock implements Serializable {
    transient volatile HashEntry<K, V>[] table;  // 每个 Segment 自己的桶数组
    // ... threshold, loadFactor 等
}

static final class HashEntry<K, V> {
    final int hash;
    final K key;
    volatile V value;       // volatile 保证可见性
    volatile HashEntry<K, V> next;  // volatile 链表指针
}
```

> [!important] 关键设计：不同 Segment 之间可以并行操作，锁只锁单个 Segment。并发度上限 = Segment 数量（默认 16）。

### 1.2 put 流程

```text
put(key, value)
  1. 计算 hash，定位 Segment（hash & segmentMask）
  2. 调用 segment.lock()      ← ReentrantLock 加锁
  3. 在 Segment 内：hash & (tab.length - 1) 定位 HashEntry 桶
  4. 遍历链表：key 相同 → 覆盖 value；没有 → 新建 HashEntry（头插法）
  5. 检查是否需要扩容（Segment 内部扩容，不影响其他 Segment）
  6. segment.unlock()
```

> [!note] JDK7 的缺点：锁粒度还是粗——一个 Segment 下有多个桶，锁一个 Segment 就锁住了所有桶。而且 Segment 数量一旦创建不能改变，并发度固定。

### 1.3 size() 方法

```text
size() 流程：
  1. 先尝试不加锁统计：遍历所有 Segment，累加 count 和 modCount
  2. 再遍历一次，比较 modCount 是否变化
  3. 如果没变化 → 统计结果准确，直接返回
  4. 如果变化了（说明统计期间有修改） → 加锁所有 Segment，强制统计
```

> [!tip] 这种"先乐观尝试，失败再加锁"的思想和 CAS 一致，尽量减少锁的使用。

## 二、JDK8：CAS + synchronized（核心）

### 2.1 数据结构

```text
ConcurrentHashMap (JDK8)
  和 HashMap 一样：Node 数组 + 铿表 + 红黑树
  不再使用 Segment！

  Node<K, V>[] table
    ├── 桶[i] = null     → 空桶
    ├── 桶[i] = Node     → 铿表（头节点）
    ├── 桶[i] = TreeBin  → 红黑树（包装节点）
    └── 桶[i] = ForwardingNode → 扩容中的占位节点（hash = -1）
```

```java
// JDK8 核心节点
static class Node<K, V> implements Map.Entry<K, V> {
    final int hash;
    final K key;
    volatile V val;        // volatile！
    volatile Node<K, V> next;  // volatile！
}

// 红黑树包装
static final class TreeBin<K, V> extends Node<K, V> {
    TreeNode<K, V> root;
    volatile TreeNode<K, V> first;
    volatile int lockState;  //读写锁状态
    // ...
}

// 扩容转发节点
static final class ForwardingNode<K, V> extends Node<K, V> {
    final Node<K, V>[] nextTable;  // 新数组引用
    // hash = -1，表示该桶正在迁移
}
```

> [!important] JDK8 摒弃了 Segment，锁粒度从"Segment级别"降到了"桶级别"——只锁一个桶的头节点，不同桶之间完全并行。

### 2.2 put 流程（重点）

```java
// JDK8 put 流程伪代码
final V putVal(K key, V value, boolean onlyIfAbsent) {
    int hash = spread(key.hashCode());      // 1. 计算 hash（扰动函数）
    for (Node<K, V>[] tab = table;;) {
        int n = tab.length;
        int i = (n - 1) & hash;             // 2. 定位桶
        Node<K, V> f = tabAt(tab, i);       //    volatile 读取桶头节点

        if (f == null) {
            // 3. 桶为空 → CAS 插入（不加锁！乐观操作）
            if (casTabAt(tab, i, null, new Node<K, V>(hash, key, value, null)))
                break;                       // CAS 成功就结束
            // CAS 失败 → 其他线程抢先插入了，自旋重试
        }
        else if (f.hash == MOVED) {
            // 4. 桶正在扩容 → 帮忙扩容（ForwardingNode）
            tab = helpTransfer(tab, f);
        }
        else {
            // 5. 桶不为空 → synchronized 锁住头节点 f
            synchronized (f) {
                // 在锁内遍历链表/树：key 相同覆盖，否则尾插法插入
                // 铿表长度 ≥ 8 且数组长度 ≥ 64 → 树化
            }
        }
    }
    addCount(1L, binCount);                  // 6. 更新计数，检查是否需要扩容
    return null;
}
```

```text
put 流程总结：
  1. 计算 hash，定位桶
  2. 桶为空   → CAS 插入（乐观，不加锁）
  3. 桶在扩容 → 协助扩容
  4. 桶不为空 → synchronized 锁头节点，遍历插入/覆盖
  5. 更新计数，触发扩容检查
```

> [!tip] 三种策略分层：空桶用 CAS（最轻量、无锁），非空桶用 synchronized（锁头节点、粒度最小），扩容时多线程协同（不阻塞读）。

### 2.3 get 流程

```java
// JDK8 get 流程——全程不加锁！
public V get(Object key) {
    Node<K, V>[] tab = table;
    Node<K, V> e = tabAt(tab, (tab.length - 1) & spread(key.hashCode()));
    // 遍历链表/树，找到就返回 val
    // val 和 next 都是 volatile，保证可见性，不需要加锁
}
```

> [!note] get 不加锁的原因：Node 的 val 和 next 都用 volatile 修饰，保证了线程间可见性。写操作也不会破坏链表结构（尾插法 + 扩容时 ForwardingNode 占位），所以读安全。

### 2.4 sizeCtl 多义字段

```text
sizeCtl 的含义取决于当前状态：

  sizeCtl > 0    → 扩容阈值（下次扩容的容量阈值，类似 HashMap 的 threshold）
  sizeCtl = 0    → 默认初始容量（未初始化时）
  sizeCtl = -1   → 正在初始化（只有一个线程能初始化，CAS 抢占）
  sizeCtl = -N   → 有 N-1 个线程正在协助扩容
                   例如 sizeCtl = -2 → 1 个线程在扩容
                   例如 sizeCtl = -3 → 2 个线程在扩容
```

> [!important] sizeCtl 是 ConcurrentHashMap 最重要的控制字段。初始化时通过 CAS 将 sizeCtl 从 0 改为 -1 来抢占初始化权，扩容时通过 CAS 将 sizeCtl 减 1 来"认领"扩容任务。

### 2.5 多线程协同扩容

```text
扩容流程：
  1. 单线程触发扩容：检查 sizeCtl，CAS 抢占扩容权
  2. 创建新数组 nextTable（2倍大小）
  3. 将扩容任务分成多个"stride"（步长区间，默认每个线程 16 个桶）
  4. 每个线程认领一段区间，从高位到低位迁移数据
  5. 迁移完一个桶，在旧数组该桶位置放 ForwardingNode（hash = MOVED = -1）
  6. 其他线程 put 时遇到 ForwardingNode → 转去协助扩容（helpTransfer）
  7. 所有桶迁移完毕 → 旧数组引用切换为新数组
```

```java
// 迁移单个桶的核心逻辑
synchronized (Node<K, V> f) {   // 锁住旧桶头节点
    // 拆分链表：高位 hash 为 0 的留在原位，高位 hash 为 1 的去新位置
    // 拆分后两组链表各挂到新数组对应位置
    // 设置旧桶为 ForwardingNode
}
```

> [!tip] 链表拆分策略和 HashMap 扩容一样：因为容量从 n 变为 2n，`hash & (2n-1)` 的高位只有 0 或 1 两种情况，链表自然分成两组，不需要重新计算 hash。

## 三、为什么不用 HashTable？

```text
HashTable（已淘汰）：
  - 每个 public 方法都加了 synchronized → 全表锁
  - 同一时刻只允许一个线程操作整张表
  - 并发性能极差，等同于串行

ConcurrentHashMap：
  - JDK7：锁粒度是 Segment（默认16段）
  - JDK8：锁粒度是桶（单个链表/树的头节点）
  - 不同桶之间完全并行，并发性能远优于 HashTable
```

| 比较维度 | HashTable | ConcurrentHashMap (JDK7) | ConcurrentHashMap (JDK8) |
|---------|-----------|---------------------------|---------------------------|
| 锁粒度 | 全表（synchronized 方法） | Segment（ReentrantLock） | 桶头节点（CAS + synchronized） |
| 并发度 | 1 | 默认16 | 理论上 = 桶数量 |
| null key/value | 不允许 | 不允许 | 不允许 |
| 继承 | Dictionary | AbstractMap | AbstractMap |
| 状态 | 已淘汰 | 较老 | 主流 |

> [!danger] HashTable 已经是遗留类（Legacy），面试提到 HashTable 时一定要说"全表锁已淘汰，用 ConcurrentHashMap 替代"。

## 原理深挖

### JDK8 为什么用 synchronized 而不是 ReentrantLock？

```text
JDK6+ synchronized 的优化链：
  无竞争 → 偏向锁（Biased Locking）：记录线程 ID，下次同线程直接进入
  有轻竞争 → 轻量锁（Thin Lock）：CAS 自旋，不升级为重量锁
  重竞争 → 重量锁（Heavy Lock）：升级为操作系统互斥量
  锁粗化（Lock Coarsening）：JIT 把相邻的锁合并成一个大锁
  锁消除（Lock Elimination）：JIT 消除不可能被共享的对象的锁
```

> [!important] 回答框架：
> 1. **性能不再是短板**：JDK6 之后 synchronized 经历了偏向锁→轻量锁→重量锁的锁升级优化，在低竞争场景和 ReentrantLock 性能接近
> 2. **JVM 层面天然支持**：synchronized 是 JVM 内置关键字，锁的升级/降级由 JVM 自动管理，不需要手动释放；ReentrantLock 是 API 层面，需要 try-finally 手动释放
> 3. **减少内存开销**：每个 ReentrantLock 需要一个 AQS 对象（包含 state + CLH 队列），而 synchronized 只需要在对象头 Mark Word 中记录锁状态，内存开销更小
> 4. **更细的锁粒度**：JDK8 CHM 锁的是桶头节点（一个 Node 对象），如果用 ReentrantLock 每个 Node 都要关联一个 AQS 对象，内存开销太大

### CAS 乐观锁的原理

```java
// CAS（Compare And Swap）——无锁乐观策略
// 原语：if (当前值 == 期望值) { 写入新值 } else { 重试 }
// 底层靠 CPU 指令 cmpxchg 保证原子性

// ConcurrentHashMap 中的 CAS 操作
static final <K, V> boolean casTabAt(Node<K, V>[] tab, int i,
                                      Node<K, V> c, Node<K, V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
// tab[i] 如果等于 c（null），就设为 v（新节点），否则返回 false → 自旋重试
```

> [!note] CAS 只适用于"竞争不激烈"的场景（空桶插入），竞争激烈时自旋开销大，此时 CHM 切换为 synchronized（悲观锁）——两种策略互补。

### volatile 在 ConcurrentHashMap 中的作用

```java
// Node 中 volatile 修饰的字段
volatile V val;           // 值可见性：写操作更新 val，get 线程立即可见
volatile Node<K, V> next; // 链表指针可见性：插入/扩容更新 next，get 立即可见

// table 数组本身的 volatile
transient volatile Node<K, V>[] table;   // 数组引用 volatile
transient volatile Node<K, V>[] nextTable; // 扩容新数组 volatile

// tabAt 用 Unsafe.getObjectVolatile 保证读到最新值
static final <K, V> Node<K, V> tabAt(Node<K, V>[] tab, int i) {
    return (Node<K, V>) U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
```

> [!important] volatile 保证了 get 操作全程无锁的安全性——写入的修改（val、next、table）对读线程立即可见，不需要 synchronized。

---

## 面试回答框架

> [!info] 问：讲一下 ConcurrentHashMap？

**① 一句话定位：** ConcurrentHashMap 是线程安全的 HashMap，JDK7 用分段锁（Segment），JDK8 用 CAS + synchronized 锁桶头节点，并发性能远优于 HashTable。

**② 核心展开：**

JDK7 的结构是 Segment 数组 + HashEntry 数组，每个 Segment 继承 ReentrantLock，不同 Segment 可以并行操作，put 时先定位 Segment 再 lock()。缺点是锁粒度还是粗——一个 Segment 下有多个桶都被锁住了。

JDK8 摒弃了 Segment，结构和 HashMap 一样（数组+链表+红黑树）。put 流程分三层策略：空桶用 CAS 直接插入（无锁最轻量）；桶正在扩容则协助扩容；桶不为空则 synchronized 锁住头节点（粒度最小只锁一个桶）。get 全程不加锁，靠 Node 的 volatile 字段保证可见性。

**③ 对比收尾：** HashTable 用全表 synchronized 方法，同一时刻只有一个线程能操作整张表，已淘汰。ConcurrentHashMap 锁粒度从 Segment 到桶头节点，越来越细，并发性能越来越好。

> [!question] 追问：JDK8 为什么用 synchronized 而不是 ReentrantLock？

> 三个原因：1) JDK6+ synchronized 做了大量优化（偏向锁/轻量锁/锁粗化），低竞争场景性能和 ReentrantLock 接近；2) synchronized 是 JVM 层面天然支持，锁升级自动管理，不需要手动释放；3) JDK8 CHM 锁的是桶头节点（Node 对象），如果用 ReentrantLock 每个 Node 都要关联一个 AQS 对象，内存开销太大。

> [!question] 追问：ConcurrentHashMap 的 key/value 能为 null 吗？

> 不能。HashMap 允许 null key 和 null value，但 ConcurrentHashMap 不允许。原因是：如果 get(key) 返回 null，在多线程环境下你无法判断是"key 不存在"还是"value 就是 null"——HashMap 可以用 containsKey 二次确认，但 ConcurrentHashMap 在并发场景下这两步操作之间可能被其他线程修改，导致语义不确定。所以设计上直接禁止 null。

---

## 跨题串联

| 引到 | 衔接 |
|------|------|
| [[../04-HashMap/HashMap详解]] | 「ConcurrentHashMap 和 HashMap JDK8 的数据结构完全相同（数组+链表+红黑树），区别只在并发控制策略——HashMap 无锁不安全，CHM 用 CAS + synchronized 保证安全」 |
| [[../../java基础/05-多线程基础/Runnable和Callable]] | 「synchronized 的锁升级过程（偏向锁→轻量锁→重量锁）是理解 CHM 为什么选择 synchronized 而不是 ReentrantLock 的关键——面试常见追问」 |
| [[../04-HashMap/HashMap详解]] | 「HashMap 的 volatile 语义和 CHM 不同——HashMap 的 table 不是 volatile，所以 HashMap 多线程 put 可能丢失数据；CHM 的 table/val/next 全 volatile，get 全程无锁也能读到最新值」 |
| [[../../java基础/01-基本语法/关键字]] | 「CAS 的底层靠 CPU cmpxchg 指令保证原子性，和 synchronized 的 Monitor 锁是两种完全不同的并发原语——乐观 vs 悲观」 |
| [[06-其他集合/其他集合速览]] | 「HashTable 是 ConcurrentHashMap 的反面教材——全表锁已淘汰，面试提到 HashTable 时一律指向 CHM」 |

---

## 记忆口诀

```
JDK7 分段锁，Segment 加锁粒度粗。
JDK8 去了 Segment，CAS 空桶乐观插，非空桶 synchronized 锁头节点。
三层策略：空桶 CAS、扩容帮忙、非空锁头。
sizeCtl 多义：正数阈值、-1 初始化、负 N 扩容 N-1 人。
volatile 保可见，get 全程不加锁。
链表拆分和 HashMap 一样，高位 0 留原位，高位 1 移新位。
HashTable 全表锁已淘汰，CHM 粒度小性能高。
synchronized 替换 ReentrantLock：锁升级优化 + JVM 天然支持 + 内存开销小。
```