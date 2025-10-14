---
tags:
  - 八股
  - java基础
  - equals
  - hashCode
status: 已完成
created: 2026-04-30
---

# equals 和 hashCode

## 一、== 和 equals 核心区别

| 比较维度 | `==` | `equals()` |
|---------|------|-----------|
| 比较对象 | 基本类型：比值<br>引用类型：比地址 | 由类自己定义比较逻辑 |
| 本质 | 运算符 | Object 类的方法 |
| 默认行为 | — | 等同于 `==`（比地址） |
| 是否可重写 | 不可 | 可重写（String/Integer 等已重写） |

```java
// == 对比
int a = 10, b = 10;
System.out.println(a == b);                         // true — 基本类型比值

String s1 = new String("abc");
String s2 = new String("abc");
System.out.println(s1 == s2);                       // false — 地址不同！

// equals 对比
System.out.println(s1.equals(s2));                  // true — String 重写了 equals 比内容
```

## 二、String 常量池陷阱

```java
String a = "abc";          // 字面量 → 常量池
String b = "abc";          // 常量池已存在 → 复用同一对象
String c = new String("abc");   // 堆中新对象
String d = new String("abc");   // 堆中又一个新对象

System.out.println(a == b);      // true  — 常量池同一对象
System.out.println(a == c);      // false — 常量池 vs 堆
System.out.println(c == d);      // false — 两个不同的堆对象

System.out.println(a.equals(c)); // true  — String.equals 比内容
System.out.println(c.equals(d)); // true  — 同上
```

> [!warning] `"字面量"` 创建的对象在常量池中会被复用，但 `new String("字面量")` 每次都在堆中创建新对象。面试区分清这两种情况。

## 三、Integer 缓存池陷阱

详见 [[包装类]]，简版回顾：

```java
Integer i1 = 127;          // valueOf(127) → 缓存池
Integer i2 = 127;          // valueOf(127) → 缓存池同一对象
System.out.println(i1 == i2);     // true

Integer i3 = 128;          // 超缓存范围 → new Integer(128)
Integer i4 = 128;          // new Integer(128) → 另一个对象
System.out.println(i3 == i4);     // false
```

## 四、hashCode 是什么

### 4.1 定义与默认实现

```java
public native int hashCode();   // Object 类的 native 方法
```

> [!abstract] hashCode 返回一个**整数值**，用于标识对象的"哈希值"。默认实现（Object 类）通常是对象内存地址的某种映射（不同 JVM 实现不同）。

### 4.2 作用

hashCode 的核心用途是**帮助哈希表快速定位**：

```text
HashMap.put("key", value) 的流程：
1. 计算 "key".hashCode() → 例如 12345
2. 取模找到桶位置 → index = 12345 % 16 = 9
3. 在桶 9 的链表/红黑树中用 equals() 精确匹配
4. 找到则更新，没找到则插入

HashMap.get("key") 的流程：
1. 计算 hashCode() → 12345
2. 定位桶 9
3. 在桶 9 中用 equals() 逐个比对 → 找到返回
```

> [!tip] hashCode 是"粗筛选"，equals 是"精确认"。先用 hashCode 快速缩小范围，再用 equals 最终确认。缺少任何一个都会导致 HashMap/HashSet 工作异常。

## 五、为什么重写 equals 必须重写 hashCode

### 5.1 场景演示

```java
public class Person {
    String name;
    int age;

    public Person(String name, int age) {
        this.name = name; this.age = age;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Person)) return false;
        Person p = (Person) o;
        return age == p.age && Objects.equals(name, p.name);
    }
    // 没有重写 hashCode()！
}
```

```java
Map<Person, String> map = new HashMap<>();
Person p1 = new Person("张三", 20);
map.put(p1, "优秀");

Person p2 = new Person("张三", 20);   // equals 与 p1 相同
System.out.println(p1.equals(p2));   // true
System.out.println(map.get(p2));     // null !!! 为什么？

// 原因：p1.hashCode() != p2.hashCode()
// → 落在不同的哈希桶 → get() 去另一个桶找，找不到
```

> [!danger] **只重写 equals 不重写 hashCode = HashMap/HashSet 出 Bug**。因为 equals 相同的两个对象 hashCode 不同，会落到不同哈希桶，HashMap 找不到。

### 5.2 正确做法

```java
@Override
public int hashCode() {
    return Objects.hash(name, age);     // JDK7+ 推荐
    // 或者手动：return 31 * name.hashCode() + age;
}

@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Person)) return false;
    Person p = (Person) o;
    return age == p.age && Objects.equals(name, p.name);
}
```

> [!tip] 使用 `Objects.hash()` 或 IDE 自动生成，不要自己拍脑袋写。IDEA 快捷键：`Alt + Insert` → `equals() and hashCode()`

## 六、equals 和 hashCode 的约定规范

### 6.1 equals 规范（自反、对称、传递、一致、非空）

| 规范 | 含义 | 示例 |
|------|------|------|
| **自反性** | `x.equals(x)` 必须返回 true | 自己等于自己 |
| **对称性** | `x.equals(y)` == `y.equals(x)` | 互相等 |
| **传递性** | `x.equals(y) && y.equals(z)` → `x.equals(z)` | A=B, B=C → A=C |
| **一致性** | 多次调用结果不变（未修改对象时） | 不能一会儿 true 一会儿 false |
| **非空性** | `x.equals(null)` 必须返回 false | 不是 null |

### 6.2 hashCode 约定

1. **同一次运行中**，同一对象多次调用 hashCode 返回相同值
2. **equals 相等 → hashCode 必须相等**（核心约定）
3. **hashCode 相等 ≠ equals 相等**（哈希碰撞是正常的）
4. **equals 不等 → hashCode 不一定不等**（但让它们不等能提高哈希表性能）

> [!summary] 核心口诀：**equals 相同则 hashCode 必须相同（同桶）；hashCode 相同 equals 不一定相同（碰撞）。**

## 原理深挖

### equals 不重写 hashCode 时 HashMap 发生了什么？

```java
class User {
    String name;
    User(String n) { this.name = n; }
    @Override public boolean equals(Object o) { return name.equals(((User)o).name); }
    // 没重写 hashCode！默认用 Object 的 native hashCode——内存地址映射
}

HashMap<User, String> map = new HashMap<>();
map.put(new User("张三"), "value");
map.get(new User("张三"));  // null！因为两个 User 的 hashCode 不同，落在不同桶
```

**执行链路：** put 时 → 调第一个 User 的 hashCode → 定位到桶 A。get 时 → 调第二个 User 的 hashCode → 定位到桶 B → 两个桶不同 → equals 根本没机会执行。

### HashMap 的 get 方法怎么用 hashCode 和 equals？

```java
// get 方法的简化逻辑
Node<K,V> get(Object key) {
    int hash = key.hashCode();           // 1. 先算 hashCode
    int index = hash & (table.length-1); // 2. 定位桶
    Node<K,V> node = table[index];       // 3. 拿到桶里的链表/树
    while (node != null) {
        if (node.hash == hash            // 4. 先比 hash（快速过滤）
            && (node.key == key || key.equals(node.key))) // 5. 再比 equals（精确确认）
            return node;
        node = node.next;
    }
    return null;
}
```

注意第 4 行：`node.hash == hash`——HashMap 内部也存了 key 的 hash 值，只比 int 非常快。hash 不等的直接跳过，hash 等的再用 equals 精确判断。

---

## 面试回答框架

> [!info] 问：== 和 equals 的区别？为什么重写 equals 必须重写 hashCode？

**① 一句话定位：** == 比栈中的值——基本类型比值，引用类型比地址。equals 看实现——Object 默认就是 ==，String 等重写了是比内容。

**② 核心展开：**

String 常量池让 `"abc" == "abc"` 为 true，但 `new String("abc") == "abc"` 为 false——因为一个在堆一个在池。Integer 缓存池让 `Integer.valueOf(127) == Integer.valueOf(127)` 为 true，但 128 就 false。

hashCode 的核心契约：equals 相等则 hashCode 必须相等。这服务于 HashMap 的工作流程——先按 hashCode 找桶做粗筛，再用 equals 做精确匹配。只重写 equals 不重写 hashCode，两个 content 相等的对象 hashCode 不同 → 落在不同桶 → HashMap 的 get 返回 null。

**③ 踩坑经验：** IDEA 生成 equals/hashCode 时选对业务字段。可变字段做 hashCode 的字段会导致 HashMap 中丢数据（key 的 hash 变了但桶位置没变）。实际开发用 Lombok 的 `@EqualsAndHashCode`。

> [!question] 追问：hashCode 相等 equals 不一定相等，这是为什么？

> 这叫哈希碰撞——不同对象的 hashCode 计算结果碰巧相同。就像不同的人可能有相同的生日。HashMap 处理碰撞的方式是拉链法：同一个桶里用链表/红黑树存多个 entry，最后靠 equals 精确匹配。

---

## 跨题串联

| 引到 | 衔接 |
|------|------|
| [[String详解]] | 「String 是 equals 和 hashCode 的最佳实践——它重写了这两个方法，所以 `"abc".equals("abc")` 永远 true，能安全做 HashMap key」 |
| [[../01-基本语法/包装类]] | 「Integer 的 equals 和 == 的差异根源在于缓存池——-128~127 范围内 == 是 true（同一对象），超出范围 == 是 false（不同对象）。但 equals 永远比内容」 |
| [[Object类方法]] | 「equals 和 hashCode 都是 Object 的方法。Object 的 hashCode 是 native 方法，返回内存地址的某种整数映射——所以两个内容相同的新对象 hashCode 不同」 |

---

## 记忆口诀

```
== 比栈值，基本类型比值，引用类型比地址。
equals 默认就是 ==，重写之后比内容。

String 常量池里复用同地址，new 出堆对象地址个个不同。
Integer 缓存一二八，范围内比 == 真，范围外比 == 假。

hashCode 做粗筛，equals 做精确认。
equals 相同必同桶（hashCode 同），同桶不一定 equals 同。
重写 equals 必重写 hashCode，否则 HashMap 找不着。
不重写的后果：两个内容相等的对象，各蹲各的桶，HashMap 喊不到。
```