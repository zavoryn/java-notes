---
tags:
  - 八股
  - 多线程
  - CAS
  - Atomic
status: 复习中
created: 2026-05-02
---

# CAS 与 Atomic 原子类

> [!summary] 我的速记
> **CAS = 乐观锁。** 比较当前值是否等于期望值 → 相等则更新，不等则重试。整个操作是一条 CPU 指令（cmpxchg），硬件保证原子性。
> **三大核心：** ABA 问题（版本号解决）、自旋开销（LongAdder 分段）、只能保证一个变量（AtomicReference 解决）。
> **AtomicInteger 源码就一行：** `value + 1`，但用 `Unsafe.compareAndSwapInt` 包了一层自旋。

---

## 一、什么是 CAS？——硬件级别的原子操作

### 1.1 定义

CAS = **Compare And Swap**（比较并交换），三个操作数：

```
CAS(V, A, B)
  V：内存地址（要操作的变量）
  A：期望的旧值
  B：要写入的新值

逻辑：
  if (V 的当前值 == A) {
      V = B;       // 相等 → 更新
      return true;
  } else {
      return false; // 不等 → 说明被其他线程改了，不更新
  }
```

### 1.2 原子性从哪来？

**CAS 不是用锁实现的，是靠 CPU 的一条原子指令。**

```java
// Java 层：AtomicInteger.incrementAndGet()
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}

// Unsafe 层：
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        v = getIntVolatile(o, offset);           // 读 volatile 最新值
    } while (!compareAndSwapInt(o, offset, v, v + delta));  // CAS 循环
    return v;
}

// Native 层（C++）：
// compareAndSwapInt → Atomic::cmpxchg → CPU 指令 cmpxchg (x86)
//   这条指令前面有 LOCK 前缀 → 锁总线/缓存行 → 保证原子性
```

> 🧠 **一句话：Java 的 CAS → Unsafe → JNI → C++ → CPU LOCK cmpxchg 指令。整个调用链自上而下，最终靠硬件保证。**

---

## 二、CAS 在 Java 中的落地——Unsafe 类

`Unsafe` 类提供了 CAS 的底层入口，三个核心方法：

```java
// 比较并替换 int
public final native boolean compareAndSwapInt(Object o, long offset, int expected, int x);

// 比较并替换 long
public final native boolean compareAndSwapLong(Object o, long offset, long expected, long x);

// 比较并替换 Object 引用
public final native boolean compareAndSwapObject(Object o, long offset, Object expected, Object x);
```

**offset 是什么？** 字段在对象内存中的偏移量，通过 `Unsafe.objectFieldOffset()` 获取。它告诉 CPU 要操作的内存地址在哪。

> 🧠 **AtomicInteger 的 value + offset：**
> ```java
> private volatile int value;                       // 当前值（volatile 保证可见性）
> private static final long valueOffset;            // value 字段的内存偏移
> static {
>     valueOffset = unsafe.objectFieldOffset
>         (AtomicInteger.class.getDeclaredField("value"));
> }
> ```

---

## 三、ABA 问题——CAS 最经典的坑

### 3.1 什么是 ABA？

```
线程 A（慢）                    线程 B（快）                     线程 C（快）
  │                               │                               │
  ├─ CAS(V, A, B)                 │                               │
  │   读 V = A                    ├─ CAS(V, A, C) 成功!           │
  │   （准备替换成 B）              │   V: A → C                   │
  │                               │                               │
  │                               ├─ CAS(V, C, A) 成功!           │
  │   ...暂停中...                 │   V: C → A                   │
  │                               │                               │
  ├─ 恢复，CAS(V, A, B) 成功!      │                               │
  │   以为 V 没变过，实际上 A→C→A 已经绕了一圈
```

> 🧠 **通俗类比：** 你出门时桌上放了 100 块钱（A），回家看到桌上还是 100 块（A），以为没人动过。实际上室友拿了 100（A→0），又放回 100（0→A）——过程中钱被用过了，但你不知道。

### 3.2 解决方案：加版本号

```java
// ❌ 普通 AtomicInteger：检测不到 ABA
AtomicInteger ai = new AtomicInteger(100);
ai.compareAndSet(100, 200);  // 线程 B 做的事

// ✅ AtomicStampedReference：加版本号（stamp）
AtomicStampedReference<Integer> asr =
    new AtomicStampedReference<>(100, 0);         // 初始值=100, 初始版本=0

// CAS 时同时比较值和版本号
int[] stampHolder = new int[1];
Integer ref = asr.get(stampHolder);               // 读值和版本
asr.compareAndSet(ref, 200,
                  stampHolder[0], stampHolder[0] + 1);  // 值从 100→200, 版本从 0→1

// ✅ AtomicMarkableReference：只关心"有没有被改过"（布尔标记）
AtomicMarkableReference<Integer> amr =
    new AtomicMarkableReference<>(100, false);
amr.compareAndSet(100, 200, false, true);         // 值从 100→200, 标记从 false→true
```

> 🧠 **选型：** `AtomicStampedReference` 用于需要精确知道被改了几次（版本号递增）。`AtomicMarkableReference` 用于只需要知道"有没有被动过"的布尔场景。

---

## 四、Atomic 类族全景

```
原子类族
  │
  ├── 基本类型
  │     ├── AtomicInteger  ← 最常用
  │     ├── AtomicLong
  │     └── AtomicBoolean
  │
  ├── 数组类型
  │     ├── AtomicIntegerArray
  │     ├── AtomicLongArray
  │     └── AtomicReferenceArray
  │
  ├── 引用类型
  │     ├── AtomicReference<V>          ← 原子更新引用
  │     ├── AtomicStampedReference<V>   ← 带版本号
  │     └── AtomicMarkableReference<V>  ← 带布尔标记
  │
  └── 累加器（JDK8+，高并发优化）
        ├── LongAdder         ← 比 AtomicLong 更高并发
        ├── DoubleAdder
        ├── LongAccumulator   ← 自定义累加函数
        └── DoubleAccumulator
```

---

## 五、LongAdder vs AtomicLong——高并发的性能差异

```
AtomicLong 的问题：
  N 个线程对同一个 value 做 CAS → 同时只有一个成功 → 其余 N-1 个失败重试
  → 高并发下 CPU 自旋空转严重

LongAdder 的解决方案——分段累加（Cell 数组）：

  LongAdder
    ├── base (基础值)
    └── Cell[] cells (分段数组)
          ├── Cell 0  ← 线程1 累加到这里
          ├── Cell 1  ← 线程2 累加到这里
          ├── Cell 2  ← 线程3 累加到这里
          └── ...

  累加时：线程先尝试 CAS base → 失败则 hash 到某个 Cell → CAS 原子累加
  取值时：sum() = base + cells[0].value + cells[1].value + ...
```

| | AtomicLong | LongAdder |
|---|---|---|
| 原理 | 单个 value，所有线程竞争 | base + Cell 数组，分散竞争 |
| 低并发 | 性能好 | 创建 Cell 有开销，略慢 |
| 高并发 | 大量 CAS 自旋，性能差 | ✅ 分段减少竞争，性能好 |
| 取值 | `get()` 精确 | `sum()` 不是原子快照（累加过程中可能不准） |
| 场景 | 低并发，需要精确当前值 | 高并发统计（QPS 计数器等） |

> 🧠 **选型一句话：** 统计计数用 LongAdder，需要精确 get 用 AtomicLong。

---

## 六、CAS 的优缺点

| 优点 | 缺点 |
|------|------|
| ✅ **无锁**，不涉及线程切换 | ❌ **自旋开销**——CAS 失败一直循环消耗 CPU |
| ✅ **无死锁**风险（没有锁） | ❌ **ABA 问题**——值没变不等于没被改过 |
| ✅ **性能好**（低竞争下） | ❌ **只能保证一个变量**的原子操作 |
| ✅ **高并发**（LongAdder 分段） | ❌ **高竞争**下不如锁（CAS 空转浪费 > 锁挂起） |

---

## 七、面试满分话术

> **面试官：** 讲讲 CAS 原理和 ABA 问题？

**你的回答：**

> **第一层（原理）：** CAS 是 Compare And Swap——硬件级别的原子操作。三个参数：内存地址 V、期望值 A、新值 B。CPU 原子地检查 V 是否等于 A，等于就更新为 B，不等就失败。底层是 CPU 的 `cmpxchg` 指令加 `LOCK` 前缀——锁总线或缓存行，硬件保证原子性。
>
> **第二层（Java 实现）：** Java 通过 Unsafe 类封装的 `compareAndSwapInt/compareAndSwapObject` 三个 native 方法调用。以 AtomicInteger 为例，`incrementAndGet()` 就是一个自旋循环——读 volatile value → CAS 期望旧值、更新为新值 → 失败就重试。value 用 volatile 修饰保证可见性。
>
> **第三层（ABA 问题）：** CAS 只能判断"值变没变"，不能判断"值变了多少次"。线程 A 读取值 A → 线程 B 把 A 改成 C 再改回 A → 线程 A 的 CAS 仍然成功，但它不知道中间经历了 A→C→A。解决方式：`AtomicStampedReference` 加版本号，每次更新版本号自增，CAS 时同时比较值和版本号。
>
> **第四层（高并发优化）：** AtomicLong 高并发下大量 CAS 自旋浪费 CPU。JDK8 引入 LongAdder，用 base + Cell 数组分段减少竞争——有点像 ConcurrentHashMap 的思路。统计计数场景推荐 LongAdder。

> **追问：CAS 和 synchronized 怎么选？**
>
> CAS 是无锁乐观策略——适合低竞争、操作极短的场景（计数器、状态标志）。synchronized 是悲观锁——适合竞争激烈或操作复杂的场景。临界区执行时间短于线程切换开销时用 CAS，否则用 synchronized。JDK 中很多类把两者结合使用——比如 ConcurrentHashMap 用 CAS 插入节点，竞争激烈时退化到 synchronized。

---

## 常见面试问题清单

| # | 问题 | 回答要点 |
|---|------|----------|
| Q1 | **CAS 是什么？底层怎么实现？** | Compare And Swap，CPU `cmpxchg` + LOCK 前缀。Java 通过 Unsafe.compareAndSwapInt 调用 |
| Q2 | **ABA 问题是什么？怎么解决？** | 值 A→C→A，CAS 只判断值不变但不知道中间变过。AtomicStampedReference 加版本号 |
| Q3 | **AtomicInteger 的 incrementAndGet 源码流程？** | 自旋循环：读 volatile value → CAS(旧值, 旧值+1) → 失败重试，直到成功 |
| Q4 | **LongAdder 为什么比 AtomicLong 快？** | 分段累加（base + Cell[]），分散竞争。高并发下线程 hash 到不同 Cell，减少 CAS 失败 |
| Q5 | **CAS 缺点有哪些？** | ①自旋空转消耗 CPU ②ABA 问题 ③只能保证一个变量原子（AtomicReference 间接解决） |
| Q6 | **CAS 和 synchronized 怎么选？** | CAS 适合低竞争+极短操作（计数器）；synchronized 适合竞争激烈或复杂逻辑；ConcurrentHashMap 两者混用 |

---

## 八、跨题串联

| 引到 | 衔接 |
|------|------|
| [[../02-synchronized锁升级/synchronized锁升级\|synchronized 锁升级]] | 「轻量级锁的 CAS 自旋和 AtomicInteger 的 CAS 循环是同源的——都是 Unsafe.compareAndSwap*」 |
| [[../01-JMM与volatile/JMM与volatile\|JMM 与 volatile]] | 「AtomicInteger 的 value 是 volatile——CAS 保证原子性，volatile 保证可见性，两者互补」 |
| [[../04-AQS与Lock/AQS与Lock\|AQS 与 Lock]] | 「AQS 的 state 就是通过 CAS 修改的——整个 Lock 体系建立在一个 CAS 变量之上」 |
| [[../06-ThreadLocal/ThreadLocal详解\|ThreadLocal]] | 「LongAdder 的 Cell 累加和 ThreadLocal 都是分而治之——分散竞争，空间换时间」 |

---

## 九、记忆口诀

```
CAS 三条数，比较再交换。
CPU 原子指令，Lock 前缀锁总线。
自旋循环不放弃，ABA 加个版本号。
Atomic 做计数，LongAdder 高并发。
无锁无死锁，但要防空转。
```
