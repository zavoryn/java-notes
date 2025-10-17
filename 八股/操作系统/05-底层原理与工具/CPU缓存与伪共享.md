---
tags:
  - 八股
  - 操作系统
  - CPU缓存
  - 伪共享
  - MESI
status: 待复习
created: 2026-05-06
---

# CPU缓存与伪共享

> [!tip] 核心速记
> **CPU缓存 = 办公桌上的东西，越近越快。**
> **伪共享 = 两人在同一长条沙发上互相折腾，其实可以各坐单人沙发。**
> **MESI = M修改E独占S共享I失效，保证多核缓存一致性。**

---

## 1. CPU 缓存层级

> **比喻**：你办公桌上的东西。
>
> - **L1 缓存** = 你手里的笔（32KB，1ns，最快）
> - **L2 缓存** = 桌上的文件架（256KB，3ns）
> - **L3 缓存** = 背后的文件柜（8MB，12ns）
> - **主内存** = 资料室的档案柜（16GB，60ns）
> - **磁盘** = 城市档案馆（1TB，10ms）
>
> 每次你找资料（CPU取数据），先从手里找，找不到才去桌上，再找不到才去柜子……越远越慢。

| 层级 | 大小 | 延迟 | 说明 |
|------|------|------|------|
| L1 | 32-64KB | ~1ns | 分数据缓存(L1d)和指令缓存(L1i)，每核独有 |
| L2 | 256-512KB | ~3ns | 每核独有 |
| L3 | 4-32MB | ~12ns | 多核共享，是核间通信的桥梁 |
| 主内存 | 8-64GB | ~60ns | 所有CPU共享 |
| SSD磁盘 | - | ~0.1ms | |
| HDD磁盘 | - | ~10ms | |

---

## 2. 缓存行 (Cache Line)

缓存不是按字节读取的，而是按 **缓存行**（通常 **64 字节**）读取。

> 比喻：你要书架上的一本书，图书管理员不是只给你那一本书，而是**把那一整排（64本）都抱过来**给你。因为根据"局部性原理"，你很可能接着要看旁边的书。

**局部性原理**：
- **空间局部性**：访问地址X后，很可能接着访问X附近的数据
- **时间局部性**：访问地址X后，短时间内很可能再次访问X

---

## 3. 伪共享 (False Sharing)

### 3.1 什么是伪共享？

**定义**：两个线程修改**不同**的变量，但这两个变量恰好落在同一缓存行中，导致缓存一致性协议（MESI）强制它们互相失效对方的缓存，性能急剧下降。

> 比喻：A和B坐在同一个长条沙发上（同一缓存行）。A想坐下去，B必须站起来；B想坐下去，A必须站起来。两人互相折腾，其实完全可以各坐各的单人沙发（不同缓存行）。

```java
// 伪共享问题示例
public class FalseSharingDemo {
    // 这两个变量很可能在同一缓存行上
    private volatile long x;  // 线程A频繁修改x
    private volatile long y;  // 线程B频繁修改y
}
```

### 3.2 伪共享的性能影响

- 正常情况：两个线程各自修改独立变量，各自缓存行独立，无冲突
- 伪共享情况：每修x一次 → x的缓存行失效 → B的缓存行也失效（因为x和y在同一行） → B重新从主内存加载 → 修改y → A的缓存行失效 → ……恶性循环

**性能下降可达 10-100 倍！**（取决于修改频率）

### 3.3 解决方案：缓存行填充 (Padding)

```java
// Java 8+ 使用 @Contended 注解（需要 -XX:-RestrictContended）
@sun.misc.Contended
private volatile long x;

// 或者手动填充，使变量独占一个缓存行（64字节 = 8个long）
private volatile long p1, p2, p3, p4, p5, p6, p7; // padding
private volatile long x;                            // 实际变量
private volatile long p8, p9, p10, p11, p12, p13, p14; // padding
```

---

## 4. MESI 缓存一致性协议详解

### 4.1 四种状态

| 状态 | 名称 | 含义 | 说明 |
|------|------|------|------|
| **M** | Modified（修改） | 该缓存行数据已被修改，**与主内存不一致** | 只有本核有该数据的最新副本，其他核的缓存行必须失效 |
| **E** | Exclusive（独占） | 该缓存行数据与主内存**一致**，且只有本核有 | 本核可以自由修改（直接变为M状态），无需通知其他核 |
| **S** | Shared（共享） | 该缓存行数据与主内存**一致**，多个核都有副本 | 本核不能直接修改，修改前需要通知其他核失效 |
| **I** | Invalid（失效） | 该缓存行**无效**，不可使用 | 需要从主内存或其他核重新加载 |

### 4.2 状态转换规则

```
核心读操作：
  I → 读请求 → 如果其他核有M状态 → 从M核获取数据+M核降为S → 本核升为S
                → 如果其他核有E/S状态 → 从主内存读取 → 本核升为S，E核降为S
                → 如果没有其他核有该数据 → 从主内存读取 → 本核升为E

核心写操作：
  E → 写 → 直接修改 → 升为M（无需通知其他核！这就是E的价值）
  S → 写 → 发出Invalidate消息通知其他核 → 其他核降为I → 本核升为M
  M → 写 → 直接修改 → 保持M（无需通知）
  I → 写 → 发出Read-Intent-to-Modify → 从M核获取+M核降为I → 本核升为M

本地核被通知：
  M → 收到Invalidate → 写回主内存 → 降为I
  M → 收到Read → 写回主内存 → 降为S
  E → 收到Read → 降为S
  S → 收到Invalidate → 降为I
```

> [!note] MESI 的核心思想
> MESI 保证：**同一时刻，只有一个核可以修改某个缓存行的数据**（M状态独占），其他核要么共享（S），要么失效（I）。
> 这是通过 **Invalidate 消息** 机制实现的——修改前必须通知其他核失效。

---

## 5. 内存屏障 (Memory Barrier) 与 volatile

### 5.1 为什么需要内存屏障？

MESI 协议保证了缓存一致性，但有两个问题：
1. **乱序执行（Out-of-Order Execution）**：CPU和编译器为了性能会重排指令顺序，可能导致程序逻辑错误
2. **MESI的Invalidate消息有延迟**：收到消息后不是立即失效，而是放入invalidate queue，后续才处理

内存屏障 = 阻止屏障前后的指令重排 + 强制刷新缓存（或等待invalidate queue处理完毕）。

### 5.2 四类内存屏障

| 类型 | 作用 | 说明 |
|------|------|------|
| **LoadLoad** | Load1; LoadLoad; Load2 → Load1必须在Load2之前完成 | 防止读重排 |
| **StoreStore** | Store1; StoreStore; Store2 → Store1必须在Store2之前完成 | 防止写重排，确保Store1刷新到缓存/主内存后再做Store2 |
| **LoadStore** | Load1; LoadStore; Store2 → Load1必须在Store2之前完成 | 防止读后写重排 |
| **StoreLoad** | Store1; StoreLoad; Load2 → Store1必须在Load2之前完成 | **最强的屏障**，防止写后读重排，同时等待invalidate queue处理完毕 |

> [!warning] StoreLoad 是最重的屏障
> StoreLoad = 既要等所有写操作刷新到主内存，又要等所有invalidate消息处理完毕。开销最大，相当于一个"全同步点"。

### 5.3 volatile 与内存屏障的关系

Java 的 `volatile` 变量的读写规则：
- **volatile 写**：前面插入 StoreStore，后面插入 StoreLoad → 确保写操作之前的所有写都已完成，且写操作之后的读能看到最新值
- **volatile 读**：后面插入 LoadLoad + LoadStore → 确保读操作之后的读写不会被重排到读之前

```
volatile写的屏障序列：
  StoreStore屏障 → volatile写 → StoreLoad屏障

volatile读的屏障序列：
  volatile读 → LoadLoad屏障 + LoadStore屏障
```

**这就是 volatile 保证可见性和有序性的底层原理！**

> [!note] volatile 不保证原子性
> volatile 只保证可见性（MESI + 内存屏障）和有序性（禁止重排），但**不保证原子性**。
> 例如 `volatile int x; x++` 仍然不安全（x++ = 读 + 加1 + 写，三步不是原子操作）。

---

## 6. 乱序执行与 MESI 的关系

CPU 为了提高性能会进行指令乱序执行（Out-of-Order Execution）：
- CPU 内部有多个执行单元，可以并行执行不相互依赖的指令
- 编译器也会做指令重排优化

**问题**：MESI 保证缓存一致性，但不保证指令执行顺序。内存屏障是用来配合 MESI 解决乱序问题的。

```
线程A写入 volatile flag = true
  → StoreStore屏障 → 写flag → StoreLoad屏障
  → 确保flag之前的所有写操作（如初始化对象）都已完成
  → 确保flag写入后，线程B读flag能立即看到

线程B读取 if (volatile flag) { use data }
  → 读flag → LoadLoad屏障 + LoadStore屏障
  → 确保读flag之后的读data不会被重排到读flag之前
```

---

## 7. Disruptor 框架原理

### 7.1 Disruptor 是什么？

Disruptor = LMAX 开源的高性能无锁队列框架，单线程每秒可处理 600万+ TPS。

### 7.2 三大核心设计

| 设计 | 原理 | 解决的问题 |
|------|------|-----------|
| **环形缓冲区（Ring Buffer）** | 固定大小数组 + 序号指针循环使用 | 避免动态扩容和GC；预分配所有Entry对象 |
| **缓存行填充 | @Contended或手动padding使关键变量独占缓存行 | 消除伪共享——head/tail/sequence各自独占缓存行 |
| **无锁并发（CAS）** | 用 CAS 更新序列号，而非Mutex加锁 | 避免锁竞争和内核态切换 |

### 7.3 Ring Buffer 详细原理

```
Ring Buffer = 固定大小数组（如2^N个Entry）
  → 生产者：CAS更新sequence，写入Entry
  → 消费者：读取sequence对应的Entry，CAS更新消费进度

  序号取模：sequence & (size - 1)  // size是2的幂次，位运算取模超快

  消费者等待策略：
    → BlockingWaitStrategy：Lock + Condition（最慢，但最安全）
    → SleepingWaitStrategy：Thread.sleep()（折中）
    → YieldingWaitStrategy：Thread.yield()（较快）
    → BusySpinWaitStrategy：自旋等待（最快，但CPU占用高）
```

### 7.4 Disruptor vs 传统阻塞队列

| 对比 | ArrayBlockingQueue | Disruptor |
|------|-------------------|-----------|
| **锁** | ReentrantLock（Mutex） | CAS无锁 |
| **伪共享** | head/tail在同一缓存行（伪共享） | 缓存行填充消除伪共享 |
| **GC** | 每次put/take创建新对象 | 预分配，零GC |
| **吞吐量** | ~5万 TPS | ~600万 TPS |

---

## 8. 和 Java 并发编程的关联

- **volatile**：写操作会强制刷新缓存行到主存（写屏障），读操作会强制从主存重新加载（读屏障）。本质上操作的是 CPU 缓存一致性。
- **CAS (Compare-And-Swap)**：底层是 CPU 的 `cmpxchg` 指令，配合 `lock` 前缀锁定缓存行（不是锁总线！）。
- **Disruptor 框架**：高性能无锁队列，核心技巧之一就是用缓存行填充消除伪共享。
- **LongAdder vs AtomicLong**：LongAdder 通过分散热点（Cell 数组）减少伪共享冲突，高并发下性能更好。

> [!note] CAS 的 `lock` 前缀锁的是缓存行，不是总线
> 旧CPU（Pentium）的 `lock` 前缀确实锁总线，但现代CPU只锁缓存行：
> - 如果数据在L1/L2缓存中 → 锁缓存行（MESI协议保证一致性）→ 其他核无法访问该缓存行
> - 如果数据不在缓存中 → 回退到锁总线（极少发生）

---

## 面试题速查

> [!example] 本模块高频面试题

1. **CPU缓存的层级？缓存行大小？**
   - L1(32KB,1ns) → L2(256KB,3ns) → L3(8MB,12ns) → 主内存(60ns)；缓存行64字节

2. **什么是伪共享？如何解决？**
   - 不同变量在同一缓存行，MESI互相失效；解决=缓存行填充(@Contended/手动padding)

3. **MESI协议详解？四种状态转换？**
   - M修改/E独占/S共享/I失效；写操作必须通知其他核失效(Invalidate消息)

4. **内存屏障和volatile的关系？**
   - volatile写=StoreStore+StoreLoad屏障，volatile读=LoadLoad+LoadStore屏障；保证可见性和有序性

5. **Disruptor框架原理？为什么比BlockingQueue快？**
   - RingBuffer+缓存行填充+CAS无锁；vs BlockingQueue的锁+伪共享+GC

6. **CAS的lock前缀锁的是总线还是缓存行？**
   - 现代CPU锁缓存行（MESI），只在数据不在缓存时回退锁总线

---

## 交叉链接

- [[02-线程同步与死锁/线程同步与死锁|线程同步与死锁]] — CAS无锁 vs Mutex锁
- [[03-内存管理/内存管理|内存管理]] — TLB是缓存的一部分，大页减少TLB miss
- [[文件系统与Linux|文件系统与Linux]] — 同在底层原理模块
- [[../操作系统总览|操作系统总览]] — 回到总览