---
tags: [八股, java基础, JDK新特性, JDK8]
status: 已完成
---

# JDK 8 核心特性

> [!summary] 我的速记
> **JDK8 六大件：** Lambda（`(参数) -> {体}`，基于函数式接口）、Stream（创建→中间操作→终止操作，惰性求值）、Optional（防 NPE，`orElseGet` 优于 `orElse`）、接口 default/static 方法、方法引用 `::`、`java.time` 新日期 API。
> **Lambda vs 匿名内部类：** Lambda 用 `invokedynamic`，不生成额外 class 文件，`this` 指向外部类。

## 1. Lambda 表达式

### 语法

```java
(参数列表) -> { 方法体 }
// 简写规则：单行可省略 {} 和 return；单参可省略 () 和类型声明
```

### 使用前提

Lambda 表达式只能用于**函数式接口**（有且仅有一个抽象方法的接口）。

```java
// 传统写法
new Thread(new Runnable() {
    @Override
    public void run() { System.out.println("run"); }
}).start();

// Lambda 写法
new Thread(() -> System.out.println("run")).start();
```

### `@FunctionalInterface` 注解

用于编译期检查接口是否符合函数式接口规范。常见函数式接口：

| 接口 | 方法签名 | 用途 |
|------|----------|------|
| `Function<T,R>` | `R apply(T t)` | 转换 |
| `Consumer<T>` | `void accept(T t)` | 消费 |
| `Supplier<T>` | `T get()` | 提供 |
| `Predicate<T>` | `boolean test(T t)` | 判断 |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | 双参数转换 |

## 2. Stream API

### 三大步骤

```
创建流 → 中间操作（惰性求值）→ 终止操作（触发计算）
```

### 创建流

```java
// 集合
list.stream();              // 顺序流
list.parallelStream();      // 并行流
// 数组
Arrays.stream(arr);
// 直接创建
Stream.of(1, 2, 3);
Stream.iterate(0, n -> n + 2);  // 无限流
```

### 中间操作（返回值仍是 Stream，惰性求值）

```java
filter(n -> n > 5)          // 过滤
map(String::length)         // 映射转换
flatMap(s -> s.stream())    // 扁平化（将嵌套集合展开）
sorted()                    // 排序
sorted(Comparator.comparing(User::getAge))
distinct()                  // 去重（基于 equals）
limit(3)                    // 截断前 N 个
skip(2)                     // 跳过前 N 个
peek(System.out::println)   // 调试（类似 forEach，但不终止流）
```

### 终止操作（触发计算）

```java
collect(Collectors.toList())    // 收集为 List
collect(Collectors.groupingBy(User::getDept))  // 分组
collect(Collectors.toMap(User::getId, Function.identity()))
forEach(System.out::println)    // 遍历
reduce(0, Integer::sum)         // 归约
count()                         // 计数
anyMatch(n -> n > 10)           // 任一匹配
allMatch(n -> n > 0)            // 全部匹配
findFirst()                     // 返回第一个元素
max(Comparator.naturalOrder())  // 最大值
```

## 3. Optional 类

> 优雅地处理 null，避免 `NullPointerException`。

### 创建 Optional

```java
Optional<String> empty = Optional.empty();           // 空 Optional
Optional<String> opt = Optional.of("hello");          // 不为 null 时用，参数为 null 会抛 NPE
Optional<String> nullable = Optional.ofNullable(x);   // 可能为 null 时用
```

### 常用方法

```java
// 获取值
opt.get()                    // 为空时抛 NoSuchElementException，谨慎使用
opt.orElse("default")        // 为空返回默认值（无论 opt 是否为空，"default" 都会先执行）
opt.orElseGet(() -> "xxx")   // 为空时惰性执行 Supplier（推荐）
opt.orElseThrow(() -> new XxException())

// 转换
opt.map(String::length)      // 映射
opt.flatMap(s -> Optional.of(s.length()))  // 避免返回嵌套 Optional

// 判断
opt.isPresent()              // 不为空
opt.ifPresent(s -> ...)      // 不为空时执行

// 过滤
opt.filter(s -> s.length() > 3)
```

> [!important] orElse vs orElseGet
> `orElse()` 参数一定会执行，`orElseGet()` 只在 Optional 为空时才执行 Supplier。
> 如果默认值需要复杂计算，优先用 `orElseGet()` 避免性能浪费。

## 4. 接口的 default 和 static 方法

```java
public interface Vehicle {
    // 抽象方法（必须实现）
    void run();

    // default 方法：有默认实现，子类可重写也可不重写
    // 用于给接口添加新方法而不破坏已有实现类（"向后兼容"）
    default void honk() {
        System.out.println("嘀嘀");
    }

    // static 方法：属于接口自身，通过接口名调用
    static Vehicle create() {
        return new Vehicle() {
            @Override
            public void run() { System.out.println("running"); }
        };
    }
}
```

> **面试要点**：default 方法是为了给接口添加新功能而不影响已有实现类（如 `Collection.forEach()`）。如果两个接口有同签名的 default 方法，实现类必须重写以消除歧义。

## 5. 方法引用（`::`）

Lambda 表达式的语法糖，当 Lambda 体只是调用一个已存在的方法时使用。

```java
// 静态方法引用
Function<String, Integer> f1 = Integer::parseInt;
// 等价 Lambda: s -> Integer.parseInt(s)

// 实例方法引用（特定对象）
String str = "hello";
Supplier<Integer> f2 = str::length;
// 等价 Lambda: () -> str.length()

// 实例方法引用（类名）
Function<String, Integer> f3 = String::length;
// 等价 Lambda: s -> s.length()

// 构造器引用
Supplier<ArrayList<String>> f4 = ArrayList::new;
// 等价 Lambda: () -> new ArrayList<>()
```

## 6. 新的日期时间 API（`java.time`）

> 线程安全、不可变、API 清晰，取代 `java.util.Date` 和 `SimpleDateFormat`。

```java
// 日期
LocalDate date = LocalDate.now();
LocalDate specific = LocalDate.of(2024, 1, 15);
date.plusDays(7);
date.getYear();
date.isAfter(otherDate);

// 时间
LocalTime time = LocalTime.now();
// 日期时间
LocalDateTime dt = LocalDateTime.now();
dt.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));

// 时间戳（带时区）
Instant instant = Instant.now();  // UTC 时间戳

// 时间间隔
Duration duration = Duration.between(time1, time2);      // 时分秒
Period period = Period.between(date1, date2);             // 年月日年月日
```

## 7. 面试话术

> "JDK 8 是 Java 历史上里程碑式的版本，核心引入了 Lambda 表达式和 Stream API，让 Java 具备了函数式编程能力。Lambda 基于函数式接口，配合 `@FunctionalInterface` 注解做编译检查。Stream 采用惰性求值 + 终止操作的两段式设计，支持串行和并行流。Optional 提供了优雅的空值处理方案，`orElseGet()` 比 `orElse()` 更推荐用于惰性计算场景。接口新增了 default 方法和 static 方法能力——default 方法主要是为了向后兼容给接口加新功能。新的 `java.time` 包下日期时间 API 是不可变且线程安全的，彻底解决了旧 API 的线程安全和设计问题。"

> "方法引用 `::` 是 Lambda 的语法糖，当 Lambda 体只是调用某个已有方法时，用方法引用更简洁。构造器引用、静态方法引用、实例方法引用三种形式要能区分。Stream 的 `flatMap` 常用于扁平化嵌套集合，`collect(Collectors.groupingBy())` 是数据分组的高频操作。"

## 8. 记忆口诀

> **Lambda 单接口，Stream 惰性走；Optional 防 NPE，日期 time 包下找。方法引用两个冒，接口 default 不强制搞。**

> [!note]- 关联笔记
> - [[JDK9-17新特性]] — JDK 8 之后的重要版本特性
> - [[../../多线程/多线程总览|多线程与并发编程]] — 并发编程深入内容
