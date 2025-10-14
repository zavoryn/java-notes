---
tags:
  - 八股
  - Java集合
  - HashMap
created: 2026-05-01
status: 已完成
---

# HashMap 详解

> **模块位置：** [[../Java集合总览|Java集合]] → 04-HashMap（核心中的核心）
> **前置知识：** [[../01-复杂度/复杂度分析|复杂度分析]]、[[../02-ArrayList/ArrayList详解|ArrayList]]、[[../03-LinkedList/LinkedList详解|LinkedList]]
> **关联题目：** [[../05-ConcurrentHashMap/ConcurrentHashMap详解|ConcurrentHashMap]]、[[../06-其他集合/其他集合详解|HashSet/LinkedHashMap]]

---

## 一、原理深挖

### 1.1 底层数据结构演进

> [!info] JDK7 → JDK8 结构变迁
>
> | 版本 | 结构 | 节点类型 | 插入方式 |
> |------|------|----------|----------|
> | **JDK7** | 数组 + 链表 | Entry<K,V> | **头插法** |
> | **JDK8** | 数组 + 链表 + 红黑树 | Node<K,V> / TreeNode<K,V> | **尾插法** |

JDK8 核心源码结构：

```java
// JDK8 HashMap.Node（链表节点）
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;  // 单向链表（只有 next，没有 prev）
}

// JDK8 HashMap.TreeNode（红黑树节点）
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // 删除时需要断开
    boolean red;
}
```

> [!important] 为什么 JDK8 要引入红黑树？
> 当 hash 碰撞严重时，链表过长 → 查找退化为 **O(n)**。
> 红黑树将查找复杂度从 **O(n) → O(logn)**，在极端场景下（如恶意构造相同 hash 的 key 进行碰撞攻击）性能提升显著。
> 8 个节点的链表查找平均 4 次，红黑树查找最多 3 次，因此阈值选 8 是折中。

### 1.2 put 流程（面试必问，逐步骤拆解）

> [!danger] 核心流程图
>
> ```
> put(key, value)
>   │
>   ├── 1. 计算 hash：h ^ (h >>> 16)
>   │
>   ├── 2. 寻址：(n-1) & hash
>   │
>   ├── 3. 桶为空？→ 直接放入新 Node
>   │
>   ├── 4. 桶不为空 → 判断节点类型
>   │     ├── 红黑树 → 调用 putTreeVal() 树插入逻辑
>   │     └── 链表 → 遍历链表
>   │           ├── key 相同（hash && equals）→ 覆盖 value
>   │           └── 遍历完无相同 key → 尾插法插入新 Node
>   │
>   ├── 5. 链表长度 ≥ 8 且 数组长度 ≥ 64 → treeifyBin() 链表转红黑树
>   │      （数组长度 < 64 → 先扩容，不转树）
>   │
>   └── 6. size++ > threshold → resize() 扩容
> ```

#### 步骤1：hash 计算——扰动函数

```java
static final int hash(Object key) {
    int h;
    // key.hashCode() 与自身高16位做异或
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

> [!tip] 为什么需要扰动？
> 寻址公式是 `(n-1) & hash`，当 n 较小时（如默认 16），`(n-1)` 只有低 4 位参与运算。
> 如果不做扰动，高位完全不影响寻址结果 → 碰撞概率大。
> 异或操作让**高16位的特征也参与到低位的寻址**中，减少碰撞。

**示例推演：**

```java
// 假设 n = 16，n-1 = 15 (二进制 0000...00001111)
// 不扰动：只有 hash 的低4位参与 & 运算
//   hash = 0b...ABCD_EFGH  → 寻址只看 EFGH 四位
//
// 扰动后：(hash ^ (hash >>> 16))
//   让高位 ABCD 也混入低位 → 寻址范围利用更充分
```

#### 步骤2：寻址——为什么用 & 而不是 %？

```java
// 寻址公式
index = (n - 1) & hash;

// 等价于 hash % n（仅当 n 是 2 的幂时成立）
// 因为 n = 2^k 时，n-1 的二进制全是 1，& 运算就是取 hash 的低 k 位
// 位运算 & 比 % 取模快得多
```

> [!important] 容量必须是 2 的幂的原因
> 1. `(n-1) & hash` 等价于 `hash % n`，位运算效率更高
> 2. `n = 2^k` 时，`(n-1)` 二进制全为 1，hash 的每一位都参与寻址 → 分布均匀、碰撞少
> 3. 如果 n 不是 2 的幂，`(n-1)` 某些位为 0，& 运算会"屏蔽" hash 的某些位 → 碰撞概率激增
>
> HashMap 通过 `tableSizeFor()` 强制将容量调整为 2 的幂：
> ```java
> static final int tableSizeFor(int cap) {
>     int n = cap - 1;
>     n |= n >>> 1;
>     n |= n >>> 2;
>     n |= n >>> 4;
>     n |= n >>> 8;
>     n |= n >>> 16;
>     return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
> }
> // 效果：任意输入 cap → 返回 ≥ cap 的最小 2 的幂
> ```

#### 步骤3~4：桶为空 vs 桶不为空

```java
// JDK8 putVal 核心逻辑（简化）
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab = table; Node<K,V> p;
    int n;
    if (tab == null || (n = tab.length) == 0)
        n = (tab = resize()).length;            // 懒初始化

    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null); // 桶为空：直接放入
    else {
        Node<K,V> e; K k;
        // 先检查桶的首节点
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;                                // key 相同：记录旧节点，后面覆盖 value
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value); // 红黑树插入
        else {
            // 链表遍历
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null); // 尾插法
                    if (binCount >= TREEIFY_THRESHOLD - 1)    // 链表长度 ≥ 8
                        treeifyBin(tab, hash);                  // 尝试转红黑树
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;                             // 找到相同 key，跳出
                p = e;
            }
        }
        if (e != null) { // key 已存在 → 覆盖 value
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            return oldValue;
        }
    }
    ++size;
    if (size > threshold)
        resize();   // 扩容
    return null;
}
```

> [!warning] key 相等的判定条件
> `hash 相同 && (key == 引用相同 || key.equals() 逻辑相同)`
> 两层判断：先比 hash（快速筛选），再比 == 或 equals（精确匹配）。
> 这也是为什么重写 equals 必须同时重写 hashCode——如果 hashCode 不一致，根本不会进入同一个桶，equals 永远不会被调用。

#### 步骤5：链表转红黑树的条件

> [!important] 双重门槛
> 链表转红黑树需要**同时满足两个条件**：
> 1. **链表长度 ≥ 8**（`TREEIFY_THRESHOLD = 8`）
> 2. **数组长度 ≥ 64**（`MIN_TREEIFY_CAPACITY = 64`）
>
> 如果链表长度 ≥ 8 但数组长度 < 64 → **先扩容**，不转树。
> 原因：数组太小时碰撞概率本身就高，扩容让元素重新分布后链表很可能变短，没必要过早转树。
>
> 红黑树退化阈值：节点数 **≤ 6**（`UNTREEIFY_THRESHOLD = 6`）时退化为链表。
> 8 和 6 之间留了间隔，避免频繁在链表和红黑树之间来回转换。

### 1.3 扩容机制

#### 基本参数

```java
// 默认值
static final int DEFAULT_INITIAL_CAPACITY = 16;     // 默认容量
static final float DEFAULT_LOAD_FACTOR = 0.75f;     // 默认加载因子
static final int MAXIMUM_CAPACITY = 1 << 30;        // 最大容量

// 关键公式
threshold = capacity * loadFactor;  // 扩容阈值
// 默认：threshold = 16 * 0.75 = 12
// 当 size > 12 时触发扩容
```

#### 扩容流程

```java
// JDK8 resize() 核心逻辑（简化）
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;

    if (oldCap > 0) {
        newCap = oldCap << 1;  // 扩容 2 倍！
        newThr = oldThr << 1;  // threshold 也翻倍
    }
    else if (oldThr > 0)
        newCap = oldThr;       // 初始化时用 threshold 作为容量
    else {
        newCap = 16;           // 全默认值
        newThr = 12;           // 16 * 0.75
    }

    Node<K,V>[] newTab = new Node[newCap];
    table = newTab;

    // 迁移旧数据到新数组
    for (int j = 0; j < oldCap; ++j) {
        Node<K,V> e;
        if ((e = oldTab[j]) != null) {
            oldTab[j] = null;
            if (e.next == null)
                // 单节点：直接重新寻址
                newTab[e.hash & (newCap - 1)] = e;
            else if (e instanceof TreeNode)
                // 红黑树：split 分裂
                ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
            else {
                // 链表：高低位分裂
                Node<K,V> loHead = null, loTail = null;  // 低位链表（原位置）
                Node<K,V> hiHead = null, hiTail = null;  // 高位链表（原位置+oldCap）
                Node<K,V> next;
                do {
                    next = e.next;
                    if ((e.hash & oldCap) == 0) {
                        // hash 的新增高位为 0 → 留在原位置
                        if (loTail == null) loHead = e;
                        else loTail.next = e;
                        loTail = e;
                    } else {
                        // hash 的新增高位为 1 → 移到原位置 + oldCap
                        if (hiTail == null) hiHead = e;
                        else hiTail.next = e;
                        hiTail = e;
                    }
                } while ((e = next) != null);

                if (loTail != null) { loTail.next = null; newTab[j] = loHead; }
                if (hiTail != null) { hiTail.next = null; newTab[j + oldCap] = hiHead; }
            }
        }
    }
    return newTab;
}
```

> [!important] JDK8 扩容优化——不再 rehash
>
> **JDK7**：扩容后重新计算每个元素的 `hash & (newCap - 1)` → 需要 rehash
>
> **JDK8**：利用扩容后 hash 多了一位参与运算的特点，元素要么**留在原位置**，要么**移到原位置 + oldCap**：
>
> ```
> 扩容前：n = 16，寻址 = hash & 15    → 只看 hash 低 4 位
> 扩容后：n = 32，寻址 = hash & 31    → 看 hash 低 5 位
>
> 多出的第 5 位（hash & 16）：
>   = 0 → 原位置不变（如原 index=3，扩容后还是 3）
>   = 1 → 原位置 + oldCap（如原 index=3，扩容后变成 3+16=19）
> ```
>
> 这样不需要重新计算 hash，只需要看 hash 的第 5 位是 0 还是 1，效率大幅提升。

### 1.4 JDK7 死循环问题

> [!danger] 多线程并发扩容 → 链表成环 → CPU 100%

**问题根源：JDK7 头插法 + rehash**

```java
// JDK7 transfer() 方法（简化）——头插法
void transfer(Entry[] newTable) {
    for (Entry<K,V> e : table) {
        while (null != e) {
            Entry<K,V> next = e.next;           // ① 记录下一个
            int i = indexFor(e.hash, newCap);   // ② 重新寻址
            e.next = newTable[i];               // ③ 头插：e.next 指向新桶头
            newTable[i] = e;                    // ④ e 成为新桶头
            e = next;                           // ⑤ 移到下一个节点
        }
    }
}
```

**死循环场景推演：**

```
线程A 和线程B 同时扩容，原链表：[3] → [7] → [5]

线程A 执行到 ① 后暂停（e=3, next=7）
线程B 完成整个 transfer → 新链表：[5] → [7] → [3]（头插法反转了顺序）

线程A 恢复执行：
  e=3, next=7（但此时 next=7 已经被线程B修改，7.next=3）
  → 处理 3：3.next = null, newTable[3] = 3
  → 处理 7：7.next = 3, newTable[3] = 7
  → 处理 3：3.next = 7  ← 此时 3.next 已经指向 7
  → 处理 7：7.next = 3  ← 形成环！3→7→3→7→...
```

> [!success] JDK8 如何解决？
> - **改用尾插法**：保持链表原有顺序，扩容时不会反转 → 多线程扩容不会成环
> - **高低位分裂**：链表元素按新增 bit 位分成两条链，分别放到低位和高位 → 不需要遍历+头插
>
> **但 JDK8 仍然不是线程安全的！**
> 尾插法解决了死循环，但仍有**数据覆盖问题**：
> 线程A 和线程B 同时 put 到同一个空桶 → 线程A 先写入，线程B 后写入覆盖线程A 的数据。
> → 需要线程安全时使用 **ConcurrentHashMap**。

### 1.5 加载因子为什么是 0.75？

> [!tip] 空间与时间的折中
>
> | 加载因子 | 效果 |
> |----------|------|
> | 太小（如 0.5） | 空间浪费严重，频繁扩容 → 内存利用率低 |
> | 太大（如 1.0） | 碰撞概率高，链表/红黑树长 → 查询慢 |
> | **0.75** | 泊松分布下，每个桶超过 8 个元素的概率 < 0.00000001，兼顾时间和空间 |
>
> 0.75 是统计学上的最优值，也是 HashMap 作者经过大量测试验证的结论。

### 1.6 红黑树退化

> [!note] 退化触发条件
> - **扩容时**：红黑树 split 后节点数 ≤ 6 → 退化为链表（`untreeify()`）
> - **删除时**：remove 后节点数 ≤ 6 → 退化
>
> 阈值选择 6 而不是 8：避免链表长度在 7~8 之间频繁切换（树化阈值 8，退化阈值 6，中间留了缓冲区间）。

### 1.7 hash 碰撞攻击

> [!warning] 安全隐患
> 如果攻击者能控制 key 的选择，故意构造大量 **hashCode 相同但 equals 不同** 的 key → 所有 key 落入同一个桶 → 链表极长 → 查询 O(n) → 服务拒绝。
>
> JDK8 红黑树将查找从 O(n) → O(logn)，有效缓解了此类攻击。
> 但根本解决方案是使用不可预测的 hash 函数（如 String 的 hash 碰撞构造已被人研究过）。

---

## 二、面试回答框架

### Q1：讲一下 HashMap？

> [!success] 回答模板（结构→流程→演进→安全）
>
> **第一层：底层结构**
> - JDK7：数组 + 链表（Entry）
> - JDK8：数组 + 链表 + 红黑树（Node/TreeNode）
> - 链表长度 ≥ 8 且数组长度 ≥ 64 时链表转红黑树，节点 ≤ 6 时退化回链表
>
> **第二层：put 流程**
> 1. 计算 hash：`(h = key.hashCode()) ^ (h >>> 16)` — 扰动函数，让高位参与寻址
> 2. 寻址：`(n-1) & hash` — 等价取模但更快（要求 n 是 2 的幂）
> 3. 桶为空 → 直接放入；桶不为空 → 判断链表/红黑树
>    - 链表：遍历比较 hash && equals，相同则覆盖 value，不同则尾插法插入
>    - 红黑树：走 putTreeVal 树插入逻辑
> 4. 插入后链表 ≥ 8 → 判断数组 ≥ 64 则转树，否则先扩容
> 5. size > threshold → 扩容（2 倍）
>
> **第三层：JDK7 vs JDK8 差异**
> - JDK7 头插法 → 多线程扩容链表成环 → CPU 100% 死循环
> - JDK8 尾插法 + 高低位分裂 → 解决了死循环
> - 但仍不线程安全（有数据覆盖问题）
>
> **第四层：线程安全**
> - HashMap 不保证线程安全，需要线程安全用 ConcurrentHashMap

### Q2：HashMap 的扩容机制？

> [!success] 回答模板
>
> 1. 触发条件：`size > threshold`（threshold = capacity * loadFactor）
> 2. 默认：容量 16，加载因子 0.75，threshold = 12
> 3. 扩容为 **2 倍**：newCap = oldCap << 1
> 4. JDK8 扩容优化：不再 rehash，而是利用 hash 新增 bit 位判断元素位置：
>    - 新增位 = 0 → 留在原位置
>    - 新增位 = 1 → 移到原位置 + oldCap
> 5. 链表按高低位分裂成两条子链，红黑树也 split 分裂

### Q3：为什么容量必须是 2 的幂？

> [!success] 回答模板
>
> 1. `(n-1) & hash` 等价于 `hash % n`，位运算比取模快
> 2. 当 n = 2^k 时，`(n-1)` 二进制全为 1 → hash 每一位都参与寻址 → 分布均匀
> 3. 如果 n 不是 2 的幂，某些位被 & 运算屏蔽 → 碰撞概率飙升
> 4. HashMap 通过 `tableSizeFor()` 将用户传入的容量强制调整为 2 的幂

### Q4：为什么用红黑树而不是其他树？

> [!success] 回答模板
>
> - AVL 树：更严格的平衡（左右子树高度差 ≤ 1），查找更快，但增删时旋转操作更多 → 频繁增删时维护成本高
> - 红黑树：宽松平衡（最长路径 ≤ 2倍最短路径），增删旋转次数少 → 更适合 HashMap 的增删频繁场景
> - HashMap 选择红黑树是**增删查的综合平衡**，AVL 更适合读多写少的搜索场景

### Q5：重写 equals 为什么必须重写 hashCode？

> [!success] 回答模板
>
> HashMap 寻址流程：先 hash 定桶 → 再 equals 精确匹配。
> 如果只重写 equals 不重写 hashCode：
> - 逻辑上"相等"的两个对象 hash 值不同 → 落入不同桶 → equals 永远不会被调用
> - 结果：map 中出现两个"相等"的 key，违反 Map 的语义约束
>
> ```java
> // 反例
> class BadKey {
>     int id;
>     public boolean equals(Object o) { return id == ((BadKey)o).id; }
>     // 没重写 hashCode！
> }
> BadKey k1 = new BadKey(1), k2 = new BadKey(1);
> map.put(k1, "A");
> map.get(k2);  // null！因为 k1 和 k2 的默认 hashCode 不同，不在同一个桶
> ```

---

## 三、跨题串联

> [!mapping] 串联图谱
>
> **HashMap** 是 Java 集合体系的枢纽，向下串联底层原理，向上串联并发与衍生集合：
>
> | 串联方向 | 关联点 | 说明 |
> |----------|--------|------|
> | → ConcurrentHashMap | 线程安全版 | HashMap 不线程安全 → ConcurrentHashMap（分段锁/CAS+synchronized） |
> | → HashSet | 基于HashMap | HashSet 内部就是 HashMap，value 用 PRESENT 占位 |
> | → equals/hashCode | key判等 | HashMap 的 key 定位依赖 hash→equals 两步，重写规则 |
> | → LinkedList | 链表Node | HashMap 链表节点 Node 与 LinkedList 的 Node 同源（单向链表） |
> | → LinkedHashMap | 双向链表+HashMap | 继承 HashMap，Node 增加 before/after 维护插入/访问顺序 |
> | → 红黑树/AVL | 树结构选择 | 为什么选红黑树不选 AVL：增删旋转少的综合平衡 |
> | → ArrayList | 数组+扩容 | HashMap 的数组扩容与 ArrayList 的数组扩容同源（2倍/1.5倍） |

```
HashMap（核心枢纽）
  │
  ├── 并发 → ConcurrentHashMap（线程安全版）
  │           ├── JDK7 Segment 分段锁
  │           └── JDK8 CAS + synchronized
  │
  ├── 衍生 → HashSet（value=PRESENT）
  │         → LinkedHashMap（before/after 维护顺序）
  │         → TreeMap（红黑树，按 key 排序）
  │
  ├── 基础 → equals/hashCode（key 定位两步判等）
  │         → 链表 Node（与 LinkedList 同源）
  │         → 红黑树 vs AVL（增删查综合平衡）
  │
  └── 机制 → 扩容（数组 2 倍，高低位分裂）
             → 扰动函数（h ^ (h >>> 16))
             → 加载因子 0.75（空间时间折中）
```

> [!tip] 面试串联话术
> "HashMap 的核心是 put 流程五步：hash扰动→位运算寻址→桶空则放/桶不空则链表遍历或树插入→链表≥8转树→扩容。JDK8 把头插改尾插解决了死循环，但仍然不是线程安全——需要并发场景就用 ConcurrentHashMap。HashMap 的链表和 LinkedList 的 Node 同源，红黑树的选择是增删查综合平衡优于 AVL。HashSet 就是 HashMap 去了 value 的壳。"

---

## 四、播面平台补充 🎧

> 以下笔记抓取自**播面**（bomianfm.com）——以播客形式讲面试题的音频平台。作为本笔记的外部视角补充。

| 笔记 | 来源 | 说明 |
|------|------|------|
| [[播面-HashMap引入红黑树的原因]] | ID:64 | 面试版速记 + 详细解答 + 思维导图 + 5道关联追问 |
| [[播面-HashMap与HashTable的区别]] | ID:54 | 7维度对比表 + 源码级解析 + 场景选择建议 + 5道关联追问 |
| [[播面-HashMap补充问答]] | 关联子题 | 10道高频面试追问（树化条件、AVL对比、Fail-Fast、2的幂设计等） |

> [!note] 播面内容与本笔记的知识点对应
>
> | 本笔记知识点 | 播面对应内容 |
> |-------------|-------------|
> | 1.1 底层数据结构演进 | → [[播面-HashMap引入红黑树的原因\|红黑树引入原因]] (ID:64) |
> | 1.5 加载因子为什么是 0.75 | → 补充问答 Q5（极端情况性能退化） |
> | 1.6 红黑树退化 | → 补充问答 Q1（树化触发条件）、Q4（链表vs红黑树性能） |
> | 1.7 hash 碰撞攻击 | → 补充问答 Q2（红黑树防碰撞攻击） |
> | Q4：为什么用红黑树而不是其他树 | → 补充问答 Q3（AVL vs 红黑树） |
> | Q5：重写 equals 必须重写 hashCode | → 补充问答 Q7（null键值设计哲学） |
> | 跨题串联→HashTable | → [[播面-HashMap与HashTable的区别\|HashMap vs HashTable]] (ID:54) |
> | 线程安全 | → 补充问答 Q6/Q8（线程安全差异、高并发选型） |
> | 容量为什么是2的幂 | → 补充问答 Q10（2的幂次方设计） |