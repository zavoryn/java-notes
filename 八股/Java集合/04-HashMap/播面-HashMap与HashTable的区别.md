---
tags:
  - 八股
  - HashMap
  - HashTable
  - 播面
source: https://www.bomianfm.com/web/question/54
created: 2026-05-02
---

# HashMap 与 HashTable 的区别

> 来源：播面 | ID: 54

---

## 面试版速记总结

`HashMap` 是 `HashTable` 的现代、非线程安全的替代品，性能更高。在现代 Java 开发中，应优先使用 `HashMap`，而 `HashTable` 因其性能问题和过时的设计，已被视为遗留类，不推荐在新代码中使用。

### 关键特性/面试要点

- **线程安全 (最重要)**：
  - `HashMap`：非线程安全。
  - `HashTable`：线程安全，其所有公开方法都由 `synchronized` 关键字修饰，导致并发性能低下（锁粒度太大，为"表级锁"）。

- **Null 值支持**：
  - `HashMap`：**允许**一个 `null` key 和多个 `null` value。
  - `HashTable`：**不允许** `key` 或 `value` 为 `null`，否则抛出 `NullPointerException`。

- **性能**：
  - `HashMap`：无锁开销，性能远高于 `HashTable`。
  - `HashTable`：由于同步锁的竞争，性能在单线程和多线程环境下都显著低于 `HashMap`。

- **继承体系**：
  - `HashMap`：继承自 `AbstractMap`，是 Java 集合框架 (JCF) 的一部分。
  - `HashTable`：继承自 `Dictionary` (一个已过时的类)，虽然后来也实现了 `Map` 接口，但其"血统"更古老。

- **迭代器 (Iterator)**：
  - `HashMap`：返回的 `Iterator` 是**快速失败 (fail-fast)** 的，在迭代时若集合被修改会抛出 `ConcurrentModificationException`。
  - `HashTable`：除了 `Iterator`，还支持一个古老的 `Enumerator` 接口，它不是 `fail-fast` 的。

- **初始容量与扩容**：
  - `HashMap`：初始容量为 16，扩容为 2 的幂次方，利于高效的位运算。
  - `HashTable`：初始容量为 11，扩容为 `old*2 + 1`。

### 总结与选择

- **单线程环境**：始终选择 `HashMap`。
- **多线程环境**：应选择 **`ConcurrentHashMap`**，它提供了更精细、性能更高的分段锁或 CAS 机制。
- **`HashTable`**：除非是维护必须使用它的古老遗留代码，否则应当完全避免使用。

---

## 思维导图

```
HashMap vs HashTable 核心对比
├── 线程安全性 (核心差异)
│   ├── HashTable: 线程安全 🔥 (全表锁/Synchronized修饰方法，并发低)
│   └── HashMap: 非线程安全 (多线程下扩容可能死循环/数据覆盖)
├── Null 值处理
│   ├── HashMap: 允许 (Key=1, Value=N)
│   └── HashTable: 不允许 🔥 (Put Null 报 NPE，避免多线程下 get 歧义)
├── 底层扩容机制 💀
│   ├── 初始容量:
│   │   ├── HashMap: 16 (懒加载)
│   │   └── HashTable: 11
│   └── 扩容逻辑:
│       ├── HashMap: 2^n (扩容 2 倍) → 利用位运算 & 代替取模，效率高
│       └── HashTable: 2n + 1 (素数) → 传统取模，减少哈希冲突
├── 迭代器机制
│   ├── HashMap: Fail-Fast 🔥 (迭代中修改抛 ConcurrentModificationException)
│   └── HashTable: Enumerator (Not Fail-Fast，历史遗留)
└── 最佳实践 (考察演进)
    ├── 单线程: HashMap
    └── 多线程: ConcurrentHashMap 🔥 (分段锁/CAS/红黑树，弃用 HashTable)
```

---

## 详细解答

> `HashMap` 和 `HashTable` 是 Java 集合框架中两种重要的键值对（Key-Value）存储结构，它们都实现了 `Map` 接口（`HashTable` 在后续版本中也实现了 `Map` 接口），但它们之间存在着显著且关键的区别。

简单来说，**`HashMap` 是 `HashTable` 的现代、轻量级、非线程安全的替代品**。在现代 Java 开发中，`HashMap` 是首选，而 `HashTable` 已被视为遗留类（Legacy Class），通常不推荐在新的代码中使用。

---

### 核心区别概览表

| 特性/维度 | HashMap | HashTable |
| :--- | :--- | :--- |
| **线程安全** | **非线程安全** | **线程安全** (方法级 `synchronized` 锁) |
| **性能** | **高** (无锁开销) | **低** (锁竞争导致性能瓶颈) |
| **Null 支持** | **允许** (一个 `null` key, 多个 `null` value) | **不允许** (key 和 value 均不能为 `null`) |
| **父类/继承体系** | `extends AbstractMap` | `extends Dictionary` (遗留类) |
| **迭代器 (Iterator)** | `Iterator` (支持 `fail-fast` 机制) | `Enumerator` / `Iterator` (`Enumerator` 不支持 `fail-fast`) |
| **初始容量与扩容** | 初始容量 **16**，扩容为 **2 的幂次方** | 初始容量 **11**，扩容为 **`old*2 + 1`** |
| **推出版本** | JDK 1.2，Java 集合框架 (JCF) 成员 | JDK 1.0，元老级类 |

---

### 详细解析

#### 1. 线程安全 (Thread Safety) - 最重要的区别

**`HashTable`**: **线程安全**。它的几乎所有公开方法（如 `get()`, `put()`, `remove()`）都使用了 `synchronized` 关键字进行修饰。这意味着在任何时刻，只允许一个线程访问 `HashTable` 的实例。
- **优点**: 在多线程环境下，无需额外同步措施即可保证数据一致性。
- **缺点**: 这种"全局锁"或"表级锁"的机制在高并发场景下性能极差。当一个线程访问 `HashTable` 时，其他所有试图访问的线程都必须等待，造成了严重的性能瓶颈。

**`HashMap`**: **非线程安全**。它的所有方法都没有进行同步控制。
- **优点**: 在单线程环境下，由于没有锁的开销，其读写性能远高于 `HashTable`。
- **缺点**: 在多线程环境下，如果多个线程同时对 `HashMap` 进行结构性修改（如添加或删除元素），可能会导致数据不一致，甚至在扩容时产生无限循环（在 JDK 1.7 及之前版本中），最终导致 CPU 100%。

**多线程环境下的正确选择**:
如果需要线程安全的 Map，不应该使用 `HashTable`，而应该使用 **`ConcurrentHashMap`**。`ConcurrentHashMap` 提供了更高效的线程安全机制（如分段锁或 CAS 操作），允许多个线程同时读写 Map 的不同部分，性能远超 `HashTable`。

或者，也可以使用 `Collections.synchronizedMap(new HashMap<>())` 来包装一个 `HashMap`，但其性能与 `HashTable` 类似，同样是全局锁，不如 `ConcurrentHashMap` 高效。

#### 2. 对 Null Key 和 Null Value 的支持

**`HashMap`**: **允许**。
- 它允许一个 `key` 为 `null`。所有 `null` key 都会被映射到哈希表的同一个位置（通常是索引 `0`）。
- 它允许多个 `value` 为 `null`。

**`HashTable`**: **不允许**。
- 无论是 `key` 还是 `value`，如果尝试存入 `null`，都会直接抛出 `NullPointerException` 异常。
- 这个设计决策是为了避免在并发环境下对 `null` 的含义产生歧义。例如，`get(key)` 返回 `null` 时，无法判断是"这个 key 不存在"还是"这个 key 对应的值就是 null"。

#### 3. 性能

- **`HashMap`**: 由于其非同步特性，在单线程环境中性能优越。
- **`HashTable`**: 由于其方法级的 `synchronized` 锁，即使在单线程环境中，每次操作也存在获取和释放锁的开销，因此性能较差。在多线程环境中，锁竞争会进一步急剧降低其性能。

**结论**: 无论单线程还是多线程，`HashMap`（或其线程安全版本 `ConcurrentHashMap`）的性能都全面优于 `HashTable`。

#### 4. 继承体系 (Inheritance Hierarchy)

- **`HashMap`**: `public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable`
  - 它继承自 `AbstractMap`，是 Java 集合框架（Java Collections Framework, JCF）的一部分，设计上更加规范和现代。

- **`HashTable`**: `public class Hashtable<K,V> extends Dictionary<K,V> implements Map<K,V>, Cloneable, java.io.Serializable`
  - 它继承自 `Dictionary` 类，这是一个在 JCF 出现之前就已经存在的抽象类，现在已经被视为过时（obsolete）。虽然为了兼容性 `HashTable` 后来也实现了 `Map` 接口，但其"血统"源自更古老的 API。

#### 5. 迭代方式 (Iterator vs Enumerator)

**`HashMap`**: `HashMap` 返回的迭代器是 `Iterator`。
- 这个 `Iterator` 是 **快速失败（fail-fast）** 的。如果在迭代过程中，有其他线程对 `HashMap` 进行了结构性修改（添加、删除元素），迭代器会立即抛出 `ConcurrentModificationException`，以防止在不确定的状态下继续操作。

**`HashTable`**: `HashTable` 不仅支持 `Iterator`，还支持一个更古老的接口 `Enumerator`（通过 `elements()` 和 `keys()` 方法获取）。
- `Enumerator` 是 **安全失败（fail-safe）** 或说非 `fail-fast` 的。它在迭代时不会抛出 `ConcurrentModificationException`。但它的问题在于，它不保证在集合被修改后迭代的准确性，可能会返回过时的数据，甚至漏掉元素，行为不可预测。

#### 6. 初始容量和扩容机制

**`HashMap`**:
- **初始容量**: 默认为 **16**。
- **扩容**: 容量总是 **2 的幂次方**。当元素数量超过 `容量 * 加载因子` 时，会创建一个新的、容量为原容量两倍的数组，并重新计算所有元素的位置（rehash）。2 的幂次方设计是为了通过位运算（`&`）代替取模（`%`）来快速计算索引，提高性能。

**`HashTable`**:
- **初始容量**: 默认为 **11**。
- **扩容**: 当元素数量超过 `容量 * 加载因子` 时，扩容后的新容量为 **`旧容量 * 2 + 1`**。这个设计导致其容量不是 2 的幂次方，索引计算通常需要使用取模运算，效率略低。

### 代码示例

```java
import java.util.HashMap;
import java.util.Hashtable;
import java.util.Map;

public class MapDifferenceDemo {
    public static void main(String[] args) {
        // 1. Null 支持对比
        Map<String, String> hashMap = new HashMap<>();
        hashMap.put("key1", "value1");
        hashMap.put(null, "null-key-value"); // 允许 null key
        hashMap.put("key2", null);            // 允许 null value
        System.out.println("HashMap with nulls: " + hashMap);

        Map<String, String> hashTable = new Hashtable<>();
        hashTable.put("key1", "value1");
        try {
            // 下面这两行都会抛出 NullPointerException
            // hashTable.put(null, "some-value");
            hashTable.put("some-key", null);
        } catch (NullPointerException e) {
            System.out.println("HashTable throws NullPointerException for null key or value.");
            e.printStackTrace(System.out);
        }
        System.out.println("HashTable: " + hashTable);
    }
}
```

### 总结与建议

| 选择 | 场景 |
| :--- | :--- |
| **`HashMap`** | **绝大多数情况下的首选**。用于单线程环境，或在多线程环境中作为局部变量使用。其性能最高，功能最符合现代编程习惯。 |
| **`ConcurrentHashMap`** | **需要线程安全的 Map 时的首选**。用于高并发服务、共享缓存等场景。它提供了卓越的并发性能和线程安全性。 |
| **`Hashtable`** | **几乎永远不要在新的代码中使用**。它的唯一用途可能是在维护那些必须与它交互的、非常古老的遗留代码或第三方库时。 |

总而言之，`HashMap` 和 `HashTable` 的区别是 Java 面试中的经典问题，它不仅考验你对基础知识的掌握，也反映了你对并发编程和 API 演进的理解。正确的认知是：**将 `HashTable` 视为历史产物，积极拥抱 `HashMap` 和 `ConcurrentHashMap`**。
