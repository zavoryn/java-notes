---
tags:
  - 八股
  - 并发编程
  - synchronized
  - 播面
source: https://www.bomianfm.com/web/question/29
created: 2026-05-02
---

# Synchronized 详解

> 来源：播面 | ID: 29

---

## 面试版速记总结

`synchronized`是Java提供的原子性、内置锁关键字，通过获取对象的监视器锁（Monitor）来实现代码块或方法的互斥访问，从而保证在多线程环境下的原子性、可见性和有序性。

### 三种使用方式与锁对象

- **修饰实例方法**：锁是当前类的实例对象 (`this`)。
- **修饰静态方法**：锁是当前类的Class对象 (`Xxx.class`)。
- **修饰代码块**：锁是括号内指定的对象，灵活性最高。

### 保证并发三大特性

- **原子性**：被`synchronized`包裹的代码块是原子操作，不可分割。
- **可见性**：JMM保证，线程释放锁前必须将共享变量的修改刷新到主内存，获取锁时会从主内存重新加载。
- **有序性**：禁止临界区（被锁代码）内外的指令重排序，保证程序串行语义。

### 关键特性

- **可重入性（Reentrancy）**：同一线程可以多次获取同一把锁而不会造成死锁。底层通过一个与锁关联的计数器实现。
- **非公平与不可中断**：是一种**非公平锁**，新来的线程可能"插队"先获得锁，可能导致线程饥饿。等待获取锁的线程是**不可中断**的。

### 与 `ReentrantLock` 对比

| 特性 | `synchronized` | `ReentrantLock` |
|:---|:---|:---|
| **本质** | Java关键字，JVM层面实现 | API类，JDK层面实现 |
| **锁管理** | 自动释放锁 | 需在`finally`块中手动调用`unlock()` |
| **功能** | 基本同步 | 支持公平/非公平选择、可中断、可超时、多Condition |

### 锁升级优化（JDK 1.6后）

1. **偏向锁**：几乎无竞争，锁偏向于第一个获取它的线程
2. **轻量级锁**：存在少量竞争，线程通过**CAS自旋**尝试获取锁
3. **重量级锁**：竞争激烈，未获取到锁的线程进入阻塞状态

---

## 思维导图

```
synchronized 🔥 (考察锁机制与核心特性)
├── 三种用法
│   ├── 实例方法 → this
│   ├── 静态方法 → Class对象
│   └── 代码块 → 指定对象
├── 并发三性
│   ├── 原子性
│   ├── 可见性
│   └── 有序性
├── 可重入性 💀 (考察计数器实现)
├── 非公平锁 💀 (考察线程饥饿风险)
├── 不可中断 💀 (考察中断响应限制)
├── 与 ReentrantLock 对比 🔥 (考察差异与选型)
│   ├── JVM vs JDK 实现
│   ├── 自动释放 vs 手动 unlock
│   └── 功能弱 vs 公平/中断/超时/多 Condition
├── 工作原理 💀 (考察 Monitor 与字节码)
│   ├── monitorenter / monitorexit
│   └── ACC_SYNCHRONIZED
├── 锁升级 🔥💀 (考察性能优化路径)
│   ├── 偏向锁 💀 (单线程低开销)
│   ├── 轻量级锁 💀 (CAS自旋少竞争)
│   └── 重量级锁 💀 (阻塞+OS调度)
└── 适用场景 🔥 (考察实际选型思路)
    ├── 冲突不极端
    ├── 无需高级特性
    └── 简洁安全优先
```

---

## 详细解答

> `synchronized` 是 Java 中用于解决多线程并发问题最基本、最常用的关键字。它的核心作用是提供一种**互斥锁（Mutual Exclusion Lock）**机制，确保在同一时刻，只有一个线程可以执行被 `synchronized` 保护的代码块或方法。

### 1. 为什么需要 `synchronized`？—— 线程安全问题

在多线程环境中，如果多个线程同时访问和修改同一个共享变量，就会出现不可预知的结果：

**原子性（Atomicity）**
- **问题**：像 `count++` 这样的操作，实际上包含三个步骤：① 读取 `count` 值；② 将值加 1；③ 将新值写回。多线程环境下，这三步之间可能被其他线程插入。
- **`synchronized` 的作用**：保证被其修饰的代码块在执行期间不会被其他线程打断，从而保证了原子性。

**可见性（Visibility）**
- **问题**：由于 Java 内存模型（JMM），每个线程有自己的工作内存（CPU 缓存），修改可能没有及时刷新到主内存。
- **`synchronized` 的作用**：释放锁时强制刷新到主内存，获取锁时强制从主内存重新加载。

**有序性（Ordering）**
- **问题**：编译器和处理器可能会对指令进行重排序。
- **`synchronized` 的作用**：隐式禁止临界区内外的指令重排序。

---

### 2. `synchronized` 的三种使用方式

#### a. 修饰实例方法

锁定的对象是**当前类的实例（`this`）**。

```java
public class MyCounter {
    private int count = 0;

    // 锁对象是 this
    public synchronized void increment() {
        count++;
    }
}

// 不同实例的同步方法互不干扰
MyCounter counter1 = new MyCounter();
MyCounter counter2 = new MyCounter();
new Thread(() -> counter1.increment()).start(); // 锁 counter1
new Thread(() -> counter2.increment()).start(); // 锁 counter2，互不影响
```

#### b. 修饰静态方法

锁定的对象是**当前类的 Class 对象（`MyCounter.class`）**，类级别的锁，所有实例共享。

```java
public class MyCounter {
    private static int count = 0;

    // 锁对象是 MyCounter.class
    public static synchronized void increment() {
        count++;
    }
}
// 所有线程竞争同一个 MyCounter.class 锁
```

#### c. 修饰代码块（最灵活）

显式指定锁对象，可以减小锁的粒度提高性能。

```java
public class MyCounter {
    private int count = 0;
    private final Object lock = new Object(); // 专用锁对象

    public void performAction() {
        // 非同步操作...
        synchronized (lock) {
            count++;  // 只锁关键代码
        }
        // 非同步操作...
    }
}
```

---

### 3. 底层原理 —— Monitor 与字节码

`synchronized` 的实现依赖于 JVM 底层的 **Monitor（监视器锁）** 机制。

- **字节码层面**：
  - 同步代码块 → 编译后生成 `monitorenter` 和 `monitorexit` 指令
  - 同步方法 → 方法访问标志中加入 `ACC_SYNCHRONIZED`

- **Monitor 内部结构**：
  - `_owner`: 指向当前持有锁的线程
  - `_EntryList`: 等待获取锁的阻塞线程队列
  - `_WaitSet`: 调用了 `wait()` 的线程队列
  - `_recursions`: 重入计数器

#### 锁升级过程

为了提高性能，`synchronized` 实现了从低到高的锁升级：

1. **偏向锁 (Biased Locking)**
   - 场景：锁大多数情况下由同一个线程获取，几乎没有竞争
   - 原理：在对象头记录线程ID，后续该线程直接进入，无需CAS
   - 升级：当其他线程尝试获取锁时，撤销偏向，升级为轻量级锁

2. **轻量级锁 (Lightweight Locking)**
   - 场景：少量线程竞争，锁持有时间短
   - 原理：线程通过 **CAS自旋** 尝试获取锁，避免线程阻塞和上下文切换
   - 升级：自旋一定次数后仍未获取到锁 → 膨胀为重量级锁

3. **重量级锁 (Heavyweight Locking)**
   - 场景：竞争激烈，或锁持有时间长
   - 原理：获取不到锁的线程被**阻塞挂起**，依赖操作系统调度，开销最大

---

### 4. 关键特性

**可重入性 (Reentrancy)**
```java
public synchronized void methodA() {
    System.out.println("进入 methodA");
    methodB(); // 同一线程可重入，不会死锁
}

public synchronized void methodB() {
    System.out.println("进入 methodB");
}
```
Monitor 内部计数器 `_recursions`：每获取一次 +1，每释放一次 -1，归零时真正释放锁。

**不可中断性**：等待 `synchronized` 锁的线程不能被 `Thread.interrupt()` 中断。

**非公平性**：新来的线程可能"插队"，不按等待队列顺序。

---

### 5. `synchronized` vs `ReentrantLock`

| 特性 | `synchronized` | `ReentrantLock` |
| :--- | :--- | :--- |
| **性质** | Java 关键字，JVM 实现 | API 类 (`java.util.concurrent.locks.Lock`) |
| **锁的释放** | **自动释放**（代码块结束/异常） | **手动释放**（`finally` 中 `unlock()`） |
| **可中断** | 不可中断 | 可中断 (`lockInterruptibly()`) |
| **公平性** | 非公平锁 | 可选公平/非公平（默认） |
| **超时获取** | 不支持 | 支持 (`tryLock(long, TimeUnit)`) |
| **绑定条件** | 只能一个条件 (`wait/notify`) | 可绑定多个 `Condition`，精确唤醒 |
| **性能** | JDK 1.6 后大幅提升 | 高竞争下精细控制更优 |

### 总结与最佳实践

1. **首选 `synchronized`**：不需要高级特性时优先使用，语法简洁，自动释放锁不易出错
2. **减小锁粒度**：尽量使用同步代码块，只包裹必要的共享资源操作
3. **使用专用锁对象**：`private final Object lock = new Object()` 而非 `this`，避免外部干扰
4. **理解锁的对象**：清楚地知道锁住的是 `this`、`.class` 还是自定义对象

---

## 关联追问（5道）

### Q1: synchronized 如何同时保证原子性和可见性？

**原子性**：进入 synchronized 块前必须获得对象锁，只有持有锁的线程能执行块内代码，期间不会被其他线程中断。

**可见性**：线程释放锁时，将工作内存中的变量刷新到主内存；线程获取锁时，清空本地缓存并从主内存重新加载变量。这形成了"锁的内存屏障"效应。

> ⚠️ 与 volatile 对比：volatile 只保证可见性和禁止指令重排序，**不保证原子性**。

---

### Q2: 修饰实例方法与静态方法的 synchronized 在锁对象上有何差异？

- **实例方法**：锁是当前调用该方法的对象实例（`this`），不同对象互不干扰 → 锁粒度细
- **静态方法**：锁是类的 Class 对象，所有实例共享同一把锁 → 锁粒度粗，全局互斥

---

### Q3: 偏向锁在什么场景下生效，又是如何升级的？

- **生效场景**：单线程重复进入同步块，几乎无并发竞争
- **升级路径**：偏向锁 → 轻量级锁 → 重量级锁
- **触发升级**：其他线程尝试获取已被偏向的锁 → 撤销偏向，CAS自旋竞争 → 自旋失败 → 膨胀为重量级锁

---

### Q4: synchronized 的可重入性是怎样通过 Monitor 实现的？

Monitor 内部维护两个关键字段：
- `owner`：记录当前持有锁的线程
- `recursionCount`：重入计数器

同一线程再次进入时，检查 `owner == 当前线程` → `recursionCount + 1`，直接进入。退出时 `recursionCount - 1`，归零时清空 `owner` 真正释放锁。

---

### Q5: 相比 ReentrantLock，synchronized 在锁释放方式上有什么特点？

**synchronized 自动释放**：进入同步块自动加锁，退出（正常结束或异常抛出）时 JVM 自动释放，避免忘记解锁导致死锁。

**ReentrantLock 手动释放**：必须在 `finally` 块中调用 `unlock()`，否则锁永不释放。
