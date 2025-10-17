---
tags:
  - 八股
  - 多线程
  - JMM
  - volatile
status: 复习中
created: 2026-05-02
---

# JMM 内存模型与 volatile 深入

> [!summary] 我的速记
> **JMM = 规范，不是实体。** 定义了线程和主内存之间的抽象关系，规定了共享变量的可见性和有序性。
> **volatile 三字诀：可见性 ✅、有序性 ✅、原子性 ❌。**
> 底层靠**内存屏障**——写屏障（StoreStore + StoreLoad）强制刷主内存，读屏障（LoadLoad + LoadStore）强制读主内存。

---

## 一、JMM（Java Memory Model）——为什么需要它？

### 1.1 问题根源：CPU 缓存与指令重排

现代 CPU 有多级缓存（L1/L2/L3），每个核心有自己的缓存，主内存是所有核心共享的。这导致了两个根本问题：

```
线程 A（CPU Core 1）          线程 B（CPU Core 2）
    │                              │
    ├── L1/L2 Cache ──┐          ├── L1/L2 Cache ──┐
    │   (x = 1)       │          │   (x = 0)       │
    │                 ↓          │                 ↓
    └──────────→ 主内存 (x = ?) ←└──────────────────┘
```

> 🧠 **核心矛盾：** CPU 缓存带来速度，但也带来了数据不一致。JMM 就是在"速度"和"一致性"之间找平衡。

### 1.2 JMM 的抽象：主内存 + 工作内存

```
线程 A                    线程 B                    线程 C
  │                         │                         │
  ├── 工作内存              ├── 工作内存              ├── 工作内存
  │   (变量副本)            │   (变量副本)            │   (变量副本)
  │                         │                         │
  └────────┬────────────────┼──────────────┬──────────┘
           │                │              │
           ▼                ▼              ▼
      ╔═══════════════════════════════════════╗
      ║          主内存 (共享变量)              ║
      ╚═══════════════════════════════════════╝
```

> 🧠 **8 种原子操作（JMM 规范）：** lock、unlock、read、load、use、assign、store、write。记住两组即可：
> - **读流程：** read(从主内存读) → load(载入工作内存) → use(线程使用)
> - **写流程：** assign(线程赋值) → store(存储到工作内存) → write(写入主内存)

---

## 二、并发编程的三大问题

| 问题 | 定义 | 产生原因 | 经典示例 |
|------|------|----------|----------|
| **可见性** | 一个线程修改了变量，其他线程看不到 | CPU 缓存——线程 A 改了缓存但没刷到主内存 | volatile 解决 |
| **原子性** | 一组操作要么全做要么全不做 | 线程切换——`i++` 是三步，中间可能被打断 | synchronized/Lock/CAS 解决 |
| **有序性** | 代码执行顺序和编写顺序不同 | 指令重排序——编译器/JVM/CPU 都可能重排 | volatile/synchronized 解决 |

```java
// 三大问题的经典案例：

// 1. 可见性问题
class VisibilityProblem {
    boolean flag = false;  // 没有 volatile！
    void writer() { flag = true; }
    void reader() { while (!flag) {} }  // 可能永远循环！线程看不到 flag 的变化
}

// 2. 原子性问题
int count = 0;
void increment() { count++; }  // 三步：读→加→写，线程切换会少加

// 3. 有序性问题（DCL 单例）
// 见下方 §5
```

---

## 三、happens-before 规则——JMM 的"因果律"

> 🧠 **一句话：happens-before 不是时间上的先后，是内存可见性的保证。** A happens-before B 意味着 A 的操作结果对 B 可见。

### 8 条核心规则（记住前 6 条够用）

| # | 规则 | 说明 | 示例 |
|---|------|------|------|
| 1 | **程序次序** | 单线程内，前面的操作 happens-before 后面的 | `x=1; y=2;` → x=1 对 y=2 可见（但可能重排序） |
| 2 | **volatile** | volatile 写 happens-before 后续 volatile 读 | 线程 A 写 volatile flag → 线程 B 读 flag 可见 |
| 3 | **锁** | unlock happens-before 后续 lock | 线程 A 释放锁 → 线程 B 获取锁可见 |
| 4 | **传递性** | A h-b B, B h-b C → A h-b C | 链式保证 |
| 5 | **start()** | 线程 A 调 `t.start()` h-b 线程 B 中任意操作 | 子线程能看到启动前的所有赋值 |
| 6 | **join()** | 线程 A 中的操作 h-b 线程 B 的 `join()` 返回 | `t.join()` 后主线程能看到 t 的所有操作 |
| 7 | 中断 | `interrupt()` h-b 被中断线程检测到中断 | — |
| 8 | 终结 | 对象构造完成 h-b `finalize()` | — |

> 🧠 **最常考的 3 条：volatile 规则、锁规则、传递性。面试时说"happens-before 保证了可见性"比说"volatile 保证可见性"更专业。**

---

## 四、volatile 深入——三大特性 + 底层实现

### 4.1 volatile 保证什么？不保证什么？

```java
volatile boolean flag = false;

// ✅ 可见性：一个线程改了 flag，另一个线程立即看到
// ✅ 有序性：flag 前后的操作不会跨越它重排序
// ❌ 原子性：flag = !flag 这种复合操作没有原子性
```

### 4.2 底层实现：内存屏障（Memory Barrier）

```
volatile 写操作：
  StoreStore 屏障  ← 禁止上面的普通写和下面的 volatile 写重排
  [volatile 写]
  StoreLoad 屏障   ← 禁止上面的 volatile 写和下面的读重排（最重的屏障）

volatile 读操作：
  LoadLoad 屏障    ← 禁止下面的普通读和上面的 volatile 读重排
  [volatile 读]
  LoadStore 屏障   ← 禁止下面的普通写和上面的 volatile 读重排
```

| 屏障类型 | 作用 | 性能消耗 |
|----------|------|----------|
| **LoadLoad** | 读-读之间不乱序 | 轻 |
| **StoreStore** | 写-写之间不乱序 | 轻 |
| **LoadStore** | 读-写之间不乱序 | 中 |
| **StoreLoad** | 写-读之间不乱序 | **最重**（需要刷缓存+同步所有核心） |

> 🧠 **StoreLoad 屏障是唯一同时具备三个功能的屏障，也是性能开销最大的——这就是 volatile 写的代价。**

### 4.3 CPU 层面的 MESI 缓存一致性协议

```
四个状态：
  M (Modified)  ← 该缓存行被修改，与主内存不一致，独占
  E (Exclusive) ← 该缓存行与主内存一致，独占
  S (Shared)    ← 该缓存行与主内存一致，可被多个核心共享
  I (Invalid)   ← 该缓存行失效

volatile 写的流程：
  1. CPU 发出 Lock 前缀指令 → 锁总线/缓存
  2. 将当前缓存行的状态改为 M
  3. 其他核心对应的缓存行 → I（失效）
  4. 写回主内存
  5. 其他核心读取时 → 缓存未命中 → 从主内存加载最新值
```

> 🧠 **面试时说"volatile 通过 Lock 前缀指令 + MESI 协议 + 内存屏障实现"就足够专业。**

---

## 五、经典应用：DCL 单例为什么要 volatile？

```java
public class Singleton {
    private static volatile Singleton instance;  // ← volatile 必须要有！

    public static Singleton getInstance() {
        if (instance == null) {                  // 第一次检查
            synchronized (Singleton.class) {
                if (instance == null) {          // 第二次检查
                    instance = new Singleton();  // ← 这行有问题！
                }
            }
        }
        return instance;
    }
}
```

**问题：`new Singleton()` 不是一个原子操作，它分三步：**

```
1. 分配内存空间
2. 初始化对象（调用构造方法、赋初始值）
3. 将 instance 引用指向内存地址

指令重排可能让 2 和 3 交换：
1. 分配内存空间
3. instance 指向内存地址  ← 重排！instance 不为 null 了
2. 初始化对象            ← 但对象还没初始化完！

结果：线程 B 在第一次检查时发现 instance != null → 直接返回 → 拿到一个半成品对象！
```

`volatile` 禁止了 2 和 3 的重排序 → 保证对象完整初始化后才能被其他线程看到。

> 🧠 **面试话术：** "DCL 单例用 volatile 不是为了可见性，而是为了**禁止指令重排**——防止 `new` 操作中分配地址和初始化对象的顺序被颠倒，导致其他线程拿到未初始化完成的半成品对象。"

---

## 六、volatile 使用场景

| 场景 | 为什么用 volatile | 示例 |
|------|-------------------|------|
| **状态标志** | 一个线程写，多个线程读 | `volatile boolean running = true;` |
| **DCL 单例** | 禁止指令重排 | `private static volatile Singleton instance;` |
| **CAS 的 value** | 保证可见性 + CAS 原子更新 | `AtomicInteger` 内部 value 是 volatile |
| **ConcurrentHashMap** | Node 的 val 和 next 是 volatile | 保证读操作的可见性 |

---

## 七、面试满分话术

> **面试官：** 讲讲 JMM 和 volatile？

**你的回答（四层递进）：**

> **第一层（JMM 定位）：** JMM 是 Java 内存模型的抽象规范，定义了线程和主内存的关系。每个线程有自己的工作内存（线程私有），所有线程共享主内存。线程操作变量时先从主内存拷贝到工作内存，修改后再刷回——这就是可见性问题的根源。
>
> **第二层（三大问题）：** 并发编程有三大核心问题——可见性、原子性、有序性。volatile 解决了前两个：通过**内存屏障**保证可见性和有序性，但不保证原子性（`i++` 这种复合操作仍需 synchronized 或 CAS）。底层是通过 Lock 前缀指令 + MESI 缓存一致性协议实现的：volatile 写会强制将缓存行刷到主内存，并使其他 CPU 的缓存行失效。
>
> **第三层（happens-before）：** JMM 用 happens-before 规则来规范可见性——volatile 写 happens-before 后续 volatile 读，锁的 unlock happens-before 后续 lock，这些规则具有传递性。面试时说到 happens-before 比只说"保证可见性"更专业。
>
> **第四层（实战 DCL）：** volatile 最经典的应用是 DCL 单例。`new Singleton()` 分三步——分配内存、初始化对象、指向引用——如果不加 volatile，2 和 3 可能被重排，其他线程拿到 instance 非 null 但实际是未初始化的半成品。这里 volatile 的核心作用是禁止指令重排，而不是可见性。

> **追问：volatile 为什么不保证原子性？**
>
> volatile 保证单个读/写操作是原子的（JVM 规范保证对 volatile 变量的单次读/写是原子的），但 `i++` 是复合操作（读→改→写），中间可能被其他线程打断。volatile 管不了这个。需要原子性就用 `AtomicInteger.incrementAndGet()`（CAS）或 `synchronized`。

---

## 常见面试问题清单

> 面试官可能从以下角度切入，每题都有对应话术：

| #   | 问题                               | 回答要点                                                                            |
| --- | -------------------------------- | ------------------------------------------------------------------------------- |
| Q1  | **什么是 JMM？**                     | 抽象规范，不是实体。主内存+工作内存，8 种原子操作（read→load→use, assign→store→write）                   |
| Q2  | **volatile 三大特性？**               | ✅可见性（Lock前缀+MESI）✅有序性（内存屏障）❌原子性（`i++`复合操作不行）                                    |
| Q3  | **volatile 底层怎么实现的？**            | 内存屏障四大件：StoreStore/StoreLoad/LoadLoad/LoadStore → CPU LOCK 指令 → MESI 协议让其他缓存行失效 |
| Q4  | **happens-before 有哪些规则？**        | 背核心 3 条：volatile 写 h-b 读、unlock h-b lock、传递性。这是可见性的规范保证                         |
| Q5  | **DCL 为什么用 volatile？**           | `new` 三步（分配内存→初始化→指向引用），2和3可重排 → 半成品对象。volatile 禁止重排                            |
| Q6  | **volatile 能替代 synchronized 吗？** | 不能！volatile 不保证原子性。复合操作必须 synchronized/CAS。volatile 适合一写多读                      |

---

## 八、跨题串联

| 引到                                                          | 衔接                                                                               |
| ----------------------------------------------------------- | -------------------------------------------------------------------------------- |
| [[../02-synchronized锁升级/synchronized锁升级\|synchronized 锁升级]] | 「synchronized 同样保证可见性和有序性——退出同步块时的 unlock 会刷主内存，底层也是内存屏障」                        |
| [[../03-CAS与Atomic/CAS与Atomic\|CAS 与 Atomic]]               | 「AtomicInteger 内部的 value 是 volatile 修饰的——CAS 是基于 volatile 可见性之上的原子操作」            |
| [[../05-线程池/线程池详解\|线程池]]                                    | 「线程池的 `shutdown()` 和 `shutdownNow()` 里用 volatile 标志位控制状态——一个线程改，其他线程见」           |
| [[../../java基础/05-多线程基础/多线程基础大纲\|多线程基础]]                    | 「sleep 和 wait 的区别：wait 释放锁，wait 被 notify 唤醒后需要重新竞争锁——竞争锁的过程中 volatie 可见性保证数据一致性」 |

---

## 九、记忆口诀

```
主内工内是抽象，数据可见靠屏障。
volatile 三特性，可见有序不原子。
写屏障刷主内存，读屏障强制刷新。
StoreLoad 最重最慢，没事别乱加 volatile。
DCL 单例防重排，半成品对象不能拿。
happens-before 八条规，传递链上保证见。
```
