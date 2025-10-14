---
tags:
  - 八股
  - Java集合
  - ArrayList
created: 2026-05-01
status: 已完成
---

# ArrayList详解

> **模块定位：** 最常用的List实现，理解动态数组是掌握所有集合的起点。
> **上一个：** [[../01-复杂度/复杂度分析|复杂度分析]]
> **下一个：** [[../03-LinkedList/LinkedList详解|LinkedList详解]]

---

## 一、原理深挖

### 1.1 底层数据结构

> [!info] 核心结构
> ArrayList底层是 **`Object[] elementData`** 动态数组。数组内存连续，通过寻址公式 `base_address + index * element_size` 直接计算内存偏移量，实现O(1)随机访问。

```java
// JDK8 ArrayList核心字段
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable {

    // 默认初始空数组（懒加载关键）
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    // 底层数组
    transient Object[] elementData; // non-private to simplify nested class access

    // 元素个数（注意：不是数组长度！）
    private int size;

    // 默认初始容量
    private static final int DEFAULT_CAPACITY = 10;
}
```

### 1.2 构造方法与懒加载

> [!question] 面试高频陷阱
> **"new ArrayList()时数组容量是10吗？"**
> → 不是！无参构造时 `elementData` 指向空数组 `{}`，**第一次add才初始化为容量10**——这是懒加载（Lazy Initialization）。

```java
// 无参构造 → 指向空数组，不分配内存
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

// 指定容量构造 → 直接分配数组
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: " + initialCapacity);
    }
}

// 传入集合构造
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    size = elementData.length;
    // ...处理返回不是Object[]的情况
}
```

> [!warning] 易错点
> `new ArrayList(10)` 只是指定了初始容量为10，**不会触发扩容**。扩容只在add时容量不足才发生。
> 懒加载的好处：避免创建后不使用时的内存浪费。

### 1.3 扩容机制

> [!info] 核心流程
> 扩容公式：`newCapacity = oldCapacity + (oldCapacity >> 1)` = **1.5倍**
> 拷贝方式：`Arrays.copyOf()` 底层调用 `System.arraycopy()`

```java
// 第一次add：空数组 → 容量10
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity); // 至少10
    }
    return minCapacity;
}

// 扩容核心方法
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1); // 1.5倍
    // >>1 即除以2，0 → 0，10 → 15，15 → 22

    // 如果1.5倍还不够（比如addAll大量元素）
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;

    // 最大容量检查
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);

    // 拷贝到新数组
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

> [!tip] 扩容细节
> - **0 → 10**：空数组第一次add，直接扩到DEFAULT_CAPACITY=10
> - **10 → 15 → 22 → 33 → ...**：每次 = old + old/2
> - **Arrays.copyOf()**：创建新数组 + System.arraycopy拷贝旧数据
> - **均摊O(1)**：扩容虽然O(n)，但发生频率递减，均摊下来每次add仍算O(1)

### 1.4 增删改查复杂度

| 操作 | 复杂度 | 原因 |
|------|--------|------|
| **get(i) / set(i)** | **O(1)** | 数组下标直接寻址 |
| **add(e) 尾插** | **O(1)均摊** | 直接放末尾；偶尔扩容O(n)均摊为O(1) |
| **add(i, e) 中间插** | **O(n)** | System.arraycopy右移i之后的元素 |
| **remove(i) 中间删** | **O(n)** | System.arraycopy左移i之后的元素 |
| **indexOf(e) 查找** | **O(n)** | 顺序遍历比较 |

```java
// 中间插入 — 需要右移元素
public void add(int index, E element) {
    rangeCheckForAdd(index);
    ensureCapacityInternal(size + 1);  // 可能扩容
    System.arraycopy(elementData, index,
                     elementData, index + 1,
                     size - index);       // 右移 [index, size-1]
    elementData[index] = element;
    size++;
}

// 中间删除 — 需要左移元素
public E remove(int index) {
    rangeCheck(index);
    E oldValue = elementData(index);
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index + 1,
                         elementData, index,
                         numMoved);         // 左移 [index+1, size-1]
    elementData[--size] = null; // 清理引用，帮助GC
    return oldValue;
}
```

> [!question] 为什么删除后要 `elementData[--size] = null`？
> → 清除末尾的冗余引用，避免对象被旧数组引用而无法被GC回收（内存泄漏）。

### 1.5 RandomAccess标记接口

```java
public interface RandomAccess {
    // 空接口，仅做标记（Marker Interface）
}
```

> [!info] 标记接口的意义
> `RandomAccess`是空接口（无方法），仅用于**标记**该集合支持高效随机访问。
> 用途：`Collections.binarySearch()`等方法会判断是否实现了RandomAccess，决定用二分查找还是遍历查找。

```java
// Collections中的判断逻辑
public static <T> int binarySearch(List<? extends Comparable<? super T>> list, T key) {
    if (list instanceof RandomAccess || list.size() < BINARYSEARCH_THRESHOLD)
        return indexedBinarySearch(list, key);   // 二分查找 O(logn)
    else
        return iteratorBinarySearch(list, key);   // 遍历查找 O(n)
}
```

### 1.6 Fail-Fast机制

> [!warning] 重点理解
> `modCount`记录集合结构修改次数。迭代器创建时保存 `expectedModCount`，迭代过程中如果 `modCount != expectedModCount`，立即抛出 **ConcurrentModificationException**。

```java
// 迭代器中的Fail-Fast检查
private class Itr implements Iterator<E> {
    int expectedModCount = modCount; // 创建时快照

    public E next() {
        checkForComodification();
        // ...
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}

// 触发场景：遍历时同时修改结构
List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c"));
for (String s : list) {
    if ("b".equals(s)) list.remove(s); // 抛ConcurrentModificationException!
}

// 正确做法：使用迭代器的remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if ("b".equals(it.next())) it.remove(); // OK，迭代器内部同步expectedModCount
}
```

### 1.7 线程安全问题

> [!danger] ArrayList线程不安全
> 多线程并发add可能导致：数据丢失、数组越界、size与elementData不一致。

**替代方案对比：**

| 方案 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| `Collections.synchronizedList(new ArrayList<>())` | 所有方法加synchronized锁 | 简单 | 全表锁，并发性能差 |
| `CopyOnWriteArrayList` | 写时复制：修改时复制新数组 | 读不加锁，读写不互斥 | 写操作昂贵O(n)，内存占用 |

```java
// 方案1：synchronizedList
List<String> syncList = Collections.synchronizedList(new ArrayList<>());
// 注意：遍历仍需手动加锁
synchronized (syncList) {
    for (String s : syncList) { ... }
}

// 方案2：CopyOnWriteArrayList
CopyOnWriteArrayList<String> cowList = new CopyOnWriteArrayList<>();
cowList.add("a"); // 内部复制整个数组 → O(n)
cowList.get(0);   // 直接读取，不加锁 → O(1)
```

### 1.8 最佳实践

> [!tip] 预估容量，避免多次扩容
> 如果已知大致元素数量，**构造时传入initialCapacity**，避免反复扩容的O(n)拷贝开销。

```java
// 不好的做法：默认容量10，add 1000个元素需多次扩容
List<String> bad = new ArrayList<>();  // 扩容路径：0→10→15→22→33→49→73→109→163→...

// 好的做法：直接指定容量
List<String> good = new ArrayList<>(1000); // 一步到位，无扩容开销
```

---

## 二、面试回答框架

### 2.1 核心问题："讲一下ArrayList底层？"

> [!success] 四维回答模板
> **① 数据结构：** 底层是Object[] elementData动态数组，内存连续，支持O(1)随机访问（寻址公式）。
> **② 扩容机制：** 无参构造指向空数组（懒加载），第一次add扩到10，之后每次1.5倍扩容（old + old>>1），用Arrays.copyOf拷贝。
> **③ 性能特性：** 查O(1)、尾插O(1)均摊、中间插删O(n)。实现了RandomAccess标记接口。
> **④ 线程安全：** 线程不安全，有Fail-Fast机制（modCount检测），替代方案用synchronizedList或CopyOnWriteArrayList。

### 2.2 追问应对

| 追问 | 回答要点 |
|------|----------|
| "new ArrayList()时容量是多少？" | 指向空数组{}，容量为0。第一次add才初始化为10——懒加载 |
| "new ArrayList(10)会扩容吗？" | 只指定了初始容量为10，不会扩容。扩容只在add时容量不足才触发 |
| "扩容比例是多少？" | 1.5倍：`newCapacity = oldCapacity + (oldCapacity >> 1)` |
| "扩容怎么拷贝数据？" | Arrays.copyOf()，底层是System.arraycopy()——native方法，效率高 |
| "ArrayList和LinkedList选哪个？" | **95%场景选ArrayList**：CPU缓存局部性（Cache Locality）好，数组内存连续对缓存友好；LinkedList每个节点独立对象，缓存命中率低 |
| "遍历时能修改吗？" | 不能直接remove/add，会抛ConcurrentModificationException。要用迭代器的remove()方法 |
| "线程安全替代？" | Collections.synchronizedList（全表锁）或CopyOnWriteArrayList（写时复制，读不加锁） |

### 2.3 ArrayList vs LinkedList对比

> [!info] 核心对比表

| 维度 | ArrayList | LinkedList |
|------|-----------|------------|
| 底层结构 | Object[] 动态数组 | Node双向链表 |
| 随机访问 | **O(1)** | O(n) |
| 头部插入 | O(n) | **O(1)** |
| 尾部插入 | O(1)均摊 | O(1) |
| 中间插入（已知位置） | O(n)搬移 | O(1)改指针 |
| 中间插入（需查找） | O(n) | **O(n)遍历+O(1)插入** |
| 内存占用 | 紧凑，无额外指针 | 每节点2个指针(prev+next)，占用更大 |
| CPU缓存 | **友好**（内存连续） | 不友好（节点分散） |
| 扩容 | 1.5倍，有拷贝开销 | 无扩容，随时加节点 |
| 适用场景 | 大多数场景 | 频繁头插/头删、不需要随机访问 |

> [!question] "95%场景选ArrayList"的理由
> 1. **CPU缓存局部性**：数组内存连续，一次缓存行(64B)加载多个元素；链表节点分散，每次访问可能miss缓存
> 2. **实际增删不如想象中慢**：System.arraycopy是native方法，内存连续时搬移很快
> 3. **内存开销小**：无额外指针开销
> 4. **现代JVM优化**：对数组操作有JIT专项优化

---

## 三、跨题串联

### 3.1 向上串联：复杂度基础

> [!example] 串联路径
> **ArrayList → 复杂度**：为什么get是O(1)？→ 数组内存连续 + 寻址公式，一步到位（回顾[[../01-复杂度/复杂度分析|复杂度分析]]的O(1)判定）。
> 为什么中间插删是O(n)？→ 一层搬移循环，与n成正比（回顾O(n)判定）。
> 均摊分析？→ 扩容虽O(n)但频率递减，整体均摊O(1)。

### 3.2 横向串联：HashMap扩容对比

> [!info] 扩容策略对比
> | 集合 | 扩容触发 | 扩容倍数 | 拷贝方式 | 何时发生 |
> |------|----------|----------|----------|----------|
> | ArrayList | size == elementData.length | **1.5倍** | Arrays.copyOf | add时容量不足 |
> | HashMap | size > capacity * loadFactor | **2倍** | 逐桶rehash | put时超过阈值 |
>
> **为什么ArrayList1.5倍、HashMap2倍？**
> - ArrayList：1.5倍是折中，兼顾内存利用率（2倍浪费较多）和扩容频率（1倍扩容太频繁）
> - HashMap：2倍保证 `hash & (n-1)` 中n始终是2的幂，掩码运算等价取模，rehash时元素位置要么不变要么偏移oldCap

### 3.3 横向串联：数组底层原理

> [!example] 为什么数组能O(1)随机访问？
> 内存中数组元素 **连续排列**，每个元素大小固定。
> 寻址公式：`address[i] = base + i * sizeof(Element)`
> CPU直接计算出地址，一步访问——不需要遍历。
> 这也是ArrayList实现RandomAccess接口的底层依据。

### 3.4 向下串联：LinkedList对比

> [!example] 串联路径
> **ArrayList → LinkedList**：同样的增删改查操作，复杂度完全不同：
> - 查：ArrayList O(1) vs LinkedList O(n) → 数组寻址公式 vs 链表遍历
> - 头插：ArrayList O(n) vs LinkedList O(1) → 数组搬移 vs 链表改指针
> - 内存：ArrayList紧凑 vs LinkedList每节点2个指针
> → 详见[[../03-LinkedList/LinkedList详解|LinkedList详解]]

### 3.5 横向串联：Fail-Fast与并发

> [!example] 串联路径
> **ArrayList Fail-Fast → ConcurrentHashMap**：
> - ArrayList用modCount做Fail-Fast检测，单线程遍历+修改时抛异常——这是"尽力而为"的检测
> - ConcurrentHashMap不做Fail-Fast，而是用弱一致性迭代器——允许遍历过程中看到部分修改
> → 详见[[../05-ConcurrentHashMap/ConcurrentHashMap详解|ConcurrentHashMap详解]]

---

## 参考来源

- 飞书文档「常见集合」(CHfMwgKfLiYDWAkvvEXc90Ycnke)
- JDK源码：`java.util.ArrayList`, `java.util.Arrays`