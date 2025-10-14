---
tags:
  - 八股
  - Java集合
  - 复习
created: 2026-05-02
reviewed: 2026-05-01
status: 已完成
---

# Java集合 总览

> **上一个模块：** [[../java基础/Java基础总览|Java基础]] ✅ 已完成（28篇，4/30）
> **当前状态：** 开始复习

---

## 一、集合体系架构

> [!info] Java 集合分为两大体系：**Collection**（单列集合）和 **Map**（双列键值对）。

### 1.1 Collection 体系

```
                          Iterable（接口）
                              │
                          Collection（接口）
                         /          \
              ┌─────────────────┐  ┌─────────────────┐
           List                 Set                 Queue/Deque
          有序可重复           无序不可重复          队列/双端队列
              │                    │                     │
    ┌────────┼────────┐    ┌──────┼──────┐    ┌─────────┼────────┐
ArrayList  LinkedList  HashSet  TreeSet    ArrayDeque LinkedList
  动态数组   双向链表   (基于HashMap)(红黑树)  循环数组   (双向链表)
  Vector           LinkedHashSet           PriorityQueue
  (已淘汰)         (HashSet+链表保序)         (二叉堆)
```

| 接口 | 特点 | 实现类 | 底层 |
|------|------|--------|------|
| **List** | 有序、可重复、有索引 | ArrayList | Object[] 动态数组 |
| | | LinkedList | 双向链表 |
| | | Vector | 数组+全表锁（已淘汰） |
| **Set** | 无序、不可重复 | HashSet | HashMap（value=PRESENT占位） |
| | | LinkedHashSet | HashSet+链表保序 |
| | | TreeSet | TreeMap（红黑树） |
| **Queue/Deque** | 队列/双端队列 | ArrayDeque | 循环数组 |
| | | PriorityQueue | 二叉堆（最小堆默认） |

### 1.2 Map 体系

```
                          Map（接口，独立体系）
                          键值对映射
                              │
              ┌───────────────┼───────────────┐
           HashMap        TreeMap         LinkedHashMap
         数组+链表+红黑树    红黑树排序     HashMap+双向链表保序
              │
        ConcurrentHashMap ← 线程安全版
        (JDK7分段锁 / JDK8 CAS+synchronized)

          HashTable ← 全表锁，已淘汰
```

| 实现类 | 底层 | 特点 | 使用场景 |
|--------|------|------|----------|
| **HashMap** | 数组+链表+红黑树 | 查O(1)平均，线程不安全 | 90%的kv场景 |
| **ConcurrentHashMap** | 同HashMap | 线程安全，JDK7分段锁→JDK8 CAS+synchronized | 多线程kv |
| **HashTable** | 数组+链表 | 全表synchronized，**已淘汰** | 不要用 |
| **LinkedHashMap** | HashMap+双向链表 | 维护插入/访问顺序 | LRU缓存 |
| **TreeMap** | 红黑树 | O(logn)，key有序 | 范围查询/排序 |

### 1.3 各集合一句话速查

| 集合 | 选它的理由 |
|------|-----------|
| **ArrayList** | 95%的List场景，查快，内存紧凑缓存友好 |
| **LinkedList** | 几乎不用，队列用ArrayDeque更好 |
| **HashSet** | 去重 |
| **TreeSet** | 排序+去重 |
| **HashMap** | 绝大部分kv场景 |
| **ConcurrentHashMap** | 多线程环境kv，替换HashTable |
| **LinkedHashMap** | LRU缓存（accessOrder=true） |
| **TreeMap** | 需要按key排序或范围查找 |
| **HashTable** | ❌ 已淘汰，用ConcurrentHashMap |
| **ArrayDeque** | 栈/队列首选，无Node开销 |
| **PriorityQueue** | Top-K问题 |

---

---

## 二、复习进度

| 序号 | 主题 | 状态 | 优先级 |
|------|------|------|--------|
| 1 | 复杂度分析 | ⬜ 待复习 | 📌 基础 |
| 2 | ArrayList | ⬜ 待复习 | 🔥 必会 |
| 3 | LinkedList | ⬜ 待复习 | 🔥 必会 |
| 4 | HashMap | ⬜ 待复习 | 🔥 核心 |
| 5 | ConcurrentHashMap | ⬜ 待复习 | 🔥 核心 |
| 6 | HashSet/HashTable/其他 | ⬜ 待复习 | 📌 重要 |

---

## 三、复习顺序

按底层依赖关系递进：
1. **复杂度分析** → 评估集合性能的基础工具
2. **ArrayList** → 最常用的 List，理解动态数组
3. **LinkedList** → 和 ArrayList 对比，理解链表
4. **HashMap** → 重中之重，数组+链表+红黑树结构
5. **ConcurrentHashMap** → HashMap 的并发版本
6. **其他集合** → HashSet/LinkedHashMap/TreeMap/HashTable

---

## 四、薄弱点标记（待温故）

> 以下是在复习过程中暴露的薄弱环节，后续需定期回顾。

| 薄弱点 | 来源模块 | 具体问题 |
|--------|----------|----------|
| Integer 缓存池 | Java基础 | 问过「127==true 128==false 为什么」，需反复记忆 `valueOf()` 源码逻辑 |
| 抽象类 vs 接口 | Java基础 | 对「接口所有方法子类都必须实现吗」理解有偏差，JDK8 default 方法改变了规则 |
| float/double 精度 | Java基础 | 提到内部类时突然问「浮点数不能表示金额的原因」，属于 BigDecimal 考点，概念串联需加强 |
| Class.forName vs loadClass | Java基础 | 对三种获取 Class 方式的适用场景理解不够，需记住 forName 执行 static 块 |
| HashMap 扰动函数 | Java集合 | 最后选中了扰动函数段落，显示对此处细节关注，需巩固 `h ^ (h >>> 16)` 原理 |
| 多线程基础 | Java基础 | 该模块仅做到了大纲级别了解，面试话术不充分，明天需深入 |
| ConcurrentHashMap sizeCtl | Java集合 | 多义字段容易混淆，需反复记忆：正数=阈值/-1=初始化/-N=N-1线程扩容 |

---

## 参考来源

- 飞书文档「常见集合」(CHfMwgKfLiYDWAkvvEXc90Ycnke)
- 飞书多维表格「Java八股复习计划」(ZyVJbwHEiaYUupsx7akca4BRnVg)
