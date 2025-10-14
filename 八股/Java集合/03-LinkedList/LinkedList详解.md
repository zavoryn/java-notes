---
tags:
  - 八股
  - Java集合
  - LinkedList
created: 2026-05-01
status: 已完成
---

# LinkedList 详解

> **模块位置：** [[../Java集合总览|Java集合]] → 03-LinkedList
> **前置知识：** [[../01-复杂度/复杂度分析|复杂度分析]]、[[../02-ArrayList/ArrayList详解|ArrayList详解]]
> **关联题目：** [[../04-HashMap/HashMap详解|HashMap]]（链表Node）、[[../06-其他集合/其他集合详解|ArrayDeque]]

---

## 一、原理深挖

### 1.1 底层数据结构：双向链表

LinkedList 底层基于**双向链表**实现，每个元素封装为一个 `Node` 对象：

```java
// JDK8 LinkedList.Node 源码（私有内部类）
private static class Node<E> {
    E item;        // 数据
    Node<E> next;  // 后继指针
    Node<E> prev;  // 前驱指针

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

> [!important] 双向链表 vs 单向链表
> - 单向链表：只有 `next`，只能从头往后遍历，删除/插入中间节点需要先找到前驱
> - 双向链表：有 `prev` + `next`，可以从任意节点向前或向后遍历，删除/插入时直接通过 `prev` 定位前驱
> - 代价：每个 Node 多存一个指针（额外 8 字节/64位系统），内存开销更大

LinkedList 同时维护头尾指针：

```java
// LinkedList 核心字段
transient Node<E> first;  // 头指针
transient Node<E> last;   // 尾指针
transient int size;       // 元素数量
```

这就意味着：**头尾操作无需遍历**，直接通过 `first` / `last` 完成。

### 1.2 时间复杂度逐项分析

| 操作 | 复杂度 | 说明 |
|------|--------|------|
| 头部增删（addFirst / removeFirst） | **O(1)** | 直接操作 `first` 指针 |
| 尾部增删（addLast / removeLast） | **O(1)** | 直接操作 `last` 指针 |
| 中间位置增删 | **O(n)** | 需先遍历定位到目标位置的前驱/后继 |
| 查找（get / contains） | **O(n)** | 不支持随机访问，必须从头或尾遍历 |
| 修改（set） | **O(n)** | 先遍历找到节点，再修改 item |

> [!warning] 常见误区："增删O(1)"
> 很多人记住"链表增删O(1)"就认为 LinkedList 增删快——但这是**删除已知节点**的复杂度。
> 实际场景中，你必须先**找到**那个节点（O(n)），然后再删除（O(1)），总复杂度是 **O(n)**。
> 只有头尾位置的增删才是真正的 O(1)。

### 1.3 源码解读：核心增删操作

**头插 addFirst：**

```java
public void addFirst(E e) {
    linkFirst(e);
}

private void linkFirst(E e) {
    final Node<E> f = first;           // 保存原头节点
    final Node<E> newNode = new Node<>(null, e, f); // 新节点.next = 原头
    first = newNode;                   // first 指向新节点
    if (f == null)
        last = newNode;                // 原链表为空，last 也指向新节点
    else
        f.prev = newNode;              // 原头.prev = 新节点
    size++;
}
```

**删除指定节点 unlink：**

```java
E unlink(Node<E> x) {
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;

    if (prev == null)        // x 是头节点
        first = next;
    else
        prev.next = next;    // 前驱跳过 x

    if (next == null)        // x 是尾节点
        last = prev;
    else
        next.prev = prev;    // 后继跳过 x

    x.item = null;           // 帮助 GC
    x.next = null;
    x.prev = null;
    size--;
    return element;
}
```

### 1.4 ArrayList vs LinkedList 全维度对比

| 对比维度 | ArrayList | LinkedList |
|----------|-----------|------------|
| **底层结构** | Object[] 动态数组 | Node 双向链表 |
| **随机查找** | O(1) — `array[index]` 直接寻址 | O(n) — 必须遍历链表 |
| **尾部增删** | O(1) — 大部分情况；扩容时 O(n) | O(1) — 直接操作 last |
| **头部增删** | O(n) — 需移动后续所有元素 | O(1) — 直接操作 first |
| **中间增删** | O(n) — 需移动元素 或 先遍历+移动 | O(n) — 需遍历定位 |
| **内存占用** | 紧凑：只存数据本身（+扩容预留空间） | 每个元素额外 2 个指针（16字节/64位） |
| **CPU缓存亲和性** | **高** — 数组内存连续，缓存行友好 | **低** — Node 散落堆中，频繁 Cache Miss |

> [!tip] CPU缓存亲和性（Cache Locality）——面试加分点
> 现代 CPU 有 L1/L2/L3 缓存，读取时按**缓存行**（通常64字节）批量加载。
> - ArrayList：元素在内存中连续排列，一次缓存行加载可命中多个元素 → **缓存友好**
> - LinkedList：每个 Node 在堆中独立分配，地址随机散落，相邻 Node 大概率不在同一缓存行 → **频繁 Cache Miss → 效率远低于理论预期**
>
> 这就是为什么 LinkedList 在实际 benchmark 中，即使理论复杂度更优（如头部增删），实际性能也往往**不如 ArrayList**。

### 1.5 实际选型结论

> [!abstract] 选型铁律
> - **95% 的场景选 ArrayList**：随机访问多、尾部增删多、内存紧凑、缓存友好
> - **LinkedList 唯一优势场景**：频繁的**头部**插入/删除（如模拟队列头部出队）
> - **队列场景推荐 ArrayDeque**：底层循环数组，无 Node 开销，头尾操作均 O(1)，性能优于 LinkedList
>
> 简记：**"除非频繁头部操作，一律 ArrayList"**

### 1.6 LinkedList 同时实现 List 和 Deque

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, Serializable
```

- 实现 `List` → 提供按索引访问的接口（但效率低）
- 实现 `Deque` → 提供双端队列操作（addFirst/addLast/removeFirst/removeLast）
- 这意味着 LinkedList 可以当作**栈**（push/pop）或**队列**（offer/poll）使用
- 但队列场景下，**ArrayDeque 性能更好**（循环数组无 Node 开销，无 GC 压力）

---

## 二、面试回答框架

### Q1：ArrayList 和 LinkedList 的区别？

> [!success] 回答模板（四维度展开法）
>
> **1. 底层结构不同**
> - ArrayList 底层是 Object[] 动态数组，内存连续
> - LinkedList 底层是双向链表，每个元素是 Node（含 prev/data/next 三字段）
>
> **2. 性能差异**
> - 查找：ArrayList O(1) 支持随机访问；LinkedList O(n) 必须遍历
> - 增删：ArrayList 头部/中间 O(n)（需移动元素）；LinkedList 头尾 O(1)、中间 O(n)
> - 实际上 LinkedList 中间增删也是 O(n)，因为要先遍历找到位置
>
> **3. 内存占用不同**
> - ArrayList 只存数据本身（扩容时有预留空间浪费）
> - LinkedList 每个 Node 额外存 2 个指针，64位系统每个元素多占 16 字节
>
> **4. CPU缓存亲和性不同（加分项）**
> - ArrayList 内存连续 → CPU 缓存行可批量加载 → 缓存友好 → 实际运行更快
> - LinkedList Node 地址散落 → 频繁 Cache Miss → 实际性能低于理论预期
>
> **结论：** 95% 场景选 ArrayList，LinkedList 仅在频繁头部操作时有优势；队列场景用 ArrayDeque 更好。

### Q2：LinkedList 增删真的是 O(1) 吗？

> [!success] 回答模板
>
> 不是。O(1) 指的是**已知节点指针时**的插入/删除操作本身。但实际使用中：
> - 调用 `add(index, element)` → 先遍历到 index 位置（O(n)），再插入（O(1）），总复杂度 **O(n)**
> - 调用 `remove(Object)` → 先遍历找到该对象（O(n)），再删除（O(1)），总复杂度 **O(n)**
> - 只有 `addFirst` / `addLast` / `removeFirst` / `removeLast` 是真正的 O(1)

### Q3：什么时候用 LinkedList？

> [!success] 回答模板
>
> 几乎不用。唯一场景是**频繁的头部插入/删除**，比如：
> - 模拟队列的头部出队操作
> - 但即使是队列场景，ArrayDeque（循环数组实现）性能更好，没有 Node 开销和 Cache Miss 问题
>
> 实际工程中 LinkedList 使用率极低，可以视为"面试专用类"。

---

## 三、跨题串联

> [!mapping] 串联图谱
>
> **LinkedList** 作为链表结构，是多个 Java 集合题目的底层组件：
>
> | 串联方向 | 关联点 | 说明 |
> |----------|--------|------|
> | → ArrayList | 对比表 | 同为 List 实现，四维度对比是高频考题 |
> | → HashMap | 链表 Node | JDK8 HashMap 的链表也是 Node 结构（prev/next/hash/key/value），与 LinkedList 的 Node 同源 |
> | → ArrayDeque | 队列替代 | 队列场景 LinkedList 被 ArrayDeque 取代（循环数组 vs 双向链表） |
> | → LinkedHashMap | 双向链表 | LinkedHashMap 在 HashMap 基础上加了双向链表维护顺序，Node 含 before/after |
> | → CPU缓存 | Cache Locality | 串联计算机体系结构：缓存行、L1/L2/L3、Cache Miss |
> | → 复杂度分析 | O(1)/O(n) | 增删 O(1) 的前提条件辨析 |

```
LinkedList
  ├── 对比 → ArrayList（四维度：结构/性能/内存/缓存）
  ├── Node → HashMap 链表节点（hash碰撞时链表结构）
  ├── Node → LinkedHashMap（before/after 维护顺序）
  ├── 替代 → ArrayDeque（队列场景更优选择）
  └── 体系 → CPU缓存亲和性（Cache Locality）
```

> [!tip] 面试串联话术
> "LinkedList 的 Node 结构和 HashMap 的链表节点本质相同——都是 prev/data/next 的组合。HashMap 处理 hash 碰撞时就是用链表串联多个 key，JDK8 链表过长会转红黑树。而 LinkedList 本身因为 Node 散落导致 Cache Miss，队列场景不如 ArrayDeque。所以 LinkedList 的核心价值在于**帮助理解链表结构**，而不是实际使用。"