---
tags:
  - 八股
  - java基础
  - String
status: 已完成
created: 2026-04-30
---

# String 详解

## 一、String 不可变性

### 1.1 底层实现

```java
// JDK 8 及之前
public final class String implements Serializable, Comparable<String>, CharSequence {
    private final char value[];   // final 修饰，数组引用不可变
}

// JDK 9+
public final class String implements Serializable, Comparable<String>, CharSequence {
    private final byte[] value;   // byte[] 替代 char[]，节省内存（Latin1 编码）
    private final byte coder;     // 标记编码：LATIN1(0) 或 UTF16(1)
}
```

> [!important] 不可变的原因：
> 1. `final class` — 类不能被继承，防止子类破坏不可变性
> 2. `private final byte[]` — 数组引用不可变，且没有暴露修改数组内容的方法
> 3. 所有修改方法（substring、replace 等）都返回 **新的 String 对象**

### 1.2 为什么设计成不可变？

1. **字符串常量池**：不可变才能安全共享
2. **HashMap 的 key**：不可变保证 hashCode 不变，否则存进去就找不到了
3. **线程安全**：天然线程安全，多线程随便用
4. **安全**：类加载器、文件路径、网络连接参数都用 String，不可变防止篡改

## 二、字符串常量池（String Pool）

### 2.1 机制

```text
+--------------------+
|   字符串常量池      |
|  (JDK7起移至堆中)   |
|                    |
|  "abc"  "hello"   |
|  "xyz"  ...       |
+--------------------+
         ↑
   字面量创建时先去常量池找，有就直接用，没有就创建
```

### 2.2 创建字符串的几种方式

```java
// 方式1：字面量 — 直接在常量池
String s1 = "abc";                        // 常量池中 "abc"

// 方式2：new — 先在常量池找/建，再在堆中新建对象
String s2 = new String("abc");            // 堆中 new，value 指向常量池

// 方式3：拼接 — 看编译器优化
String s3 = "a" + "b";                    // 编译优化 → "ab"，常量池
String s4 = s1 + "d";                     // 变量拼接 → new StringBuilder → 堆中新对象

// 方式4：intern() — 手动入常量池
String s5 = new String("abc").intern();   // 返回常量池中的引用
```

### 2.3 JDK 版本演进

| JDK 版本    | 常量池位置                     |
| --------- | ------------------------- |
| JDK 6 及以前 | 方法区（永久代 PermGen）          |
| JDK 7     | 移到堆中（Heap）                |
| JDK 8+    | 堆中的元空间（Metaspace，但常量池仍在堆） |

> [!note] JDK 7 将常量池移到堆中，解决了 PermGen 空间不足的 OOM 问题。因为字符串常量池可以享受 GC 的回收了。

## 三、intern() 方法

```java
String s1 = new String("hello");   // 创建堆对象，常量池已有 "hello"
String s2 = s1.intern();           // 返回常量池中 "hello" 的引用
System.out.println(s1 == s2);      // false，s1 是堆对象，s2 是常量池对象

String s3 = new StringBuilder().append("hello").append("world").toString();
String s4 = s3.intern();           // s3 是堆中 "helloworld"，常量池没有，intern 会把 s3 的引用存入常量池
System.out.println(s3 == s4);      // true（JDK7+，JDK6 是 false）
```

> [!tip] JDK 7+ 的 intern() 行为：如果常量池已有，直接返回池中引用；如果没有，**将堆中对象的引用复制到常量池**，而不是复制整个对象。

## 四、String / StringBuffer / StringBuilder 对比

| 比较维度 | String | StringBuffer | StringBuilder |
|---------|--------|-------------|--------------|
| 可变性 | 不可变 | 可变 | 可变 |
| 线程安全 | 安全（不可变天然安全） | 安全（synchronized） | 不安全 |
| 性能 | 拼接时慢（产生新对象） | 中（同步开销） | 快（无锁） |
| 适用场景 | 值不变、少量拼接 | 多线程大量拼接 | 单线程大量拼接 |
| 继承结构 | `final class` | `AbstractStringBuilder` | `AbstractStringBuilder` |

```java
// 单线程用 StringBuilder
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 100; i++) {
    sb.append(i);           // 同一对象追加，高效
}
String result = sb.toString();

// 多线程用 StringBuffer
StringBuffer sbf = new StringBuffer();
sbf.append("thread-safe");   // 方法有 synchronized
```

## 五、字符串拼接底层原理

### 5.1 + 号编译后的样子

```java
// 源码
String s = "hello" + " " + "world";

// 编译后（字面量拼接，编译期合并）
String s = "hello world";   // 直接常量池一个对象
```

```java
// 源码（有变量）
String s1 = "hello";
String s2 = s1 + " world";

// 编译后
String s2 = new StringBuilder()
                    .append(s1)
                    .append(" world")
                    .toString();
```

### 5.2 循环中的坑

```java
// 错误写法 — 每次循环创建 StringBuilder + String 对象
String result = "";
for (int i = 0; i < 10000; i++) {
    result += i;    // 相当于 result = new StringBuilder(result).append(i).toString();
}

// 正确写法 — 只有一个 StringBuilder
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.append(i);
}
String result = sb.toString();
```

> [!danger] 循环中禁止用 `+=` 拼接字符串，会产生大量临时对象，导致频繁 GC。

## 六、经典面试题

### 题目 1：new String("abc") 创建了几个对象？

```java
String s = new String("abc");
```

**答**：创建了 **2 个对象**。
- 第一个：**常量池**中的 `"abc"`（类加载时/字面量首次出现时创建）
- 第二个：**堆**中的 `new String("abc")` 对象（value 属性指向常量池的 char[]）

> 如果常量池已经存在 `"abc"`（比如之前已经有代码用过），就只创建 **1 个**堆对象。

### 题目 2：拼接判等

```java
String s2 = new StringBuilder().append("ja").append("va").toString();
String s3 = s2.intern();
String s1 = "java";   // 注意 "java" 是关键字，常量池可能已有
System.out.println(s2 == s3);  // ?
System.out.println(s3 == s1);  // ?
```

**答**：
- `s2 == s3`：**true**（JDK7+）。s2 在堆中，`"java"` 关键字已在常量池，所以 intern 返回常量池已有引用。如果 `"java"` 换成别的如 `"helloworld"`，没有提前入池的话，s3 就是 s2 的引用。
- `s3 == s1`：**true**，s3 是常量池引用，s1 也是常量池引用。

> [!tip] 此题核心考点：JDK7+ intern() 的"引用入池"机制 vs 关键字提前入池。

### 题目 3：三个对象

```java
String s1 = "a";                  // 1 个："a" 入常量池
String s2 = new String("a");      // 2 个：常量池 "a"（已有则不重复）+ 堆中对象
String s3 = "a" + "b";            // 1 个："ab"（编译期合并入常量池）
```

> 三个 String 对象：`"a"` 常量池、堆中的 `new String("a")`、`"ab"` 常量池。

## 原理深挖

### JDK9 为什么把 char[] 改成 byte[]？

JVM 用统计数据分析发现，绝大多数 String 的内容都是 Latin-1 字符（单字节），用 `char[]`（每个 char 占 2 字节）浪费了一半内存。JDK9 引入紧凑字符串（Compact Strings），底层用 `byte[]` + 编码标识（coder 字段：0=LATIN1，1=UTF16），Latin-1 场景节省 50% 内存。

### `+` 拼接的底层真相

```java
String s = "a" + "b" + "c";
// 编译器直接优化为：String s = "abc";（常量折叠，编译期完成）

String a = "a"; String b = "b";
String s = a + b;
// 编译器翻译为：new StringBuilder().append(a).append(b).toString();
// 每次执行都 new 一个 StringBuilder 对象！
```

这就是循环里用 `+` 的性能灾难——每次迭代 new 一个 StringBuilder。

### intern() 在 JDK7 的行为变化

JDK6：`intern()` 把对象**复制**一份到永久代的常量池（双重存储，容易 OOM）。
JDK7+：常量池移到堆里，`intern()` 直接把**堆中对象的引用**存入常量池。所以 `new String("abc").intern() == new String("abc").intern()` 为 true，且 `"abc" == new String("abc").intern()` 也为 true。

---

## 面试回答框架

> [!info] 问：说说 String、StringBuilder、StringBuffer？

**① 一句话定位：** String 不可变，StringBuilder 可变+无锁+高性能，StringBuffer 可变+有锁+线程安全。

**② 核心展开：**

String 的不可变性来自两个设计：final class 防止继承破坏契约，`private final byte[]` 防止被替换引用。设计成不可变有四个原因：常量池复用、HashMap key 安全性、线程安全、类加载器安全性。

字符串拼接方面：字面量拼接由编译器在编译期合并（常量折叠）；变量拼接编译为 `new StringBuilder().append().toString()`。循环中用 `+` 每次迭代都 new 一个 StringBuilder，性能最差。

**③ 避坑指南：** `new String("abc")` 可能创建 1-2 个对象（常量池已有就不新建）；`intern()` 用于节省内存和锁场景，但滥用会撑爆常量池；`+` 拼接在循环外（常量折叠）没问题，循环内用 StringBuilder。

---

## 跨题串联

| 引到 | 衔接 |
|------|------|
| [[equals和hashCode]] | 「String 重写了 equals 和 hashCode——所以 `"abc".equals("abc")` 永远 true，这也是 String 能安全做 HashMap key 的底层保证」 |
| [[../03-核心机制/泛型]] | 「泛型擦除后运行时类型丢失，但 `String.class` 是个例外——String 作为最常用类型，JVM 对它做了大量特殊优化」 |
| [[../01-基本语法/包装类]] | 「String 常量池和 Integer 缓存池原理相通——都是空间换时间，复用不可变对象避免重复创建」 |
| [[Object类方法]] | 「String 对 Object 的 toString/hashCode/equals 都做了重写，是 Object 方法重写的最佳实践范例」 |

---

## 记忆口诀

```
String 底层 final 不可变，JDK9 后用 byte 存。
常量池在堆里管，字面量直接池里占，new 了还得堆里见。
String 不变安全慢，Buffer 锁多线程，Builder 无锁单线程快。
字面量拼接编译期合并，变量拼接运行期 new 出 StringBuilder。
循环远离 +=，要拼就用 Builder。
intern 往常量池搬，JDK7 后搬引用不搬对象。
```