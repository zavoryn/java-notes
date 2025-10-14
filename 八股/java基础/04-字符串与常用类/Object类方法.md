---
tags:
  - 八股
  - java基础
  - Object
status: 已完成
created: 2026-04-30
---

# Object 类方法

> [!summary] 我的速记
> **9 个方法：** getClass / hashCode / equals / clone / toString / notify / notifyAll / wait(×3) / finalize
> **口诀：** 类哈等克字，通通等废（getClass/hashCode/equals/clone/toString/notify/notifyAll/wait/finalize）
> **重点：** wait/notify 必须在 `synchronized` 里；finalize 已弃用；clone 是浅拷贝。

## 一、方法总览

Object 是所有 Java 类的**终极父类**，共有 11 个方法（JDK8 统计）：

```java
public class Object {
    // 1. 反射相关
    public final native Class<?> getClass();

    // 2. 哈希相关
    public native int hashCode();              // 默认转为内存地址的映射

    // 3. 比较相关
    public boolean equals(Object obj);          // 默认 == 比地址

    // 4. 克隆
    protected native Object clone() throws CloneNotSupportedException;

    // 5. 打印
    public String toString();                   // 默认: 类名@十六进制hashCode

    // 6. 线程通信（wait/notify）
    public final native void notify();
    public final native void notifyAll();
    public final native void wait(long timeout) throws InterruptedException;
    public final void wait(long timeout, int nanos) throws InterruptedException;
    public final void wait() throws InterruptedException;

    // 7. 垃圾回收
    protected void finalize() throws Throwable;  // JDK9 @Deprecated

    // 8. 不常用（复制/构造函数相关）
    // registerNatives() 等 native 方法
}
```

## 二、逐方法详解

### 2.1 getClass()

```java
// 获取运行时类型
Object obj = new String("hello");
System.out.println(obj.getClass());         // class java.lang.String
System.out.println(obj.getClass().getName());   // java.lang.String
```

> [!tip] `getClass()` 返回的是**运行时类型**，不是编译期类型。反射的基础。

### 2.2 hashCode()

详见 [[equals和hashCode]]。默认是 native 方法，返回对象内存地址的整数映射。

- 必须与 equals 保持一致
- HashMap/HashSet 快速定位的基石

### 2.3 equals(Object obj)

详见 [[equals和hashCode]]。默认实现：

```java
public boolean equals(Object obj) {
    return (this == obj);   // 默认就是 ==，比地址
}
```

必须在子类中重写来实现"内容比较"。

### 2.4 clone()

```java
// 必须实现 Cloneable 接口，否则抛 CloneNotSupportedException
public class Person implements Cloneable {
    String name;

    @Override
    public Person clone() {
        try {
            return (Person) super.clone();   // 默认浅拷贝
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
}
```

> [!warning] **浅拷贝**：`clone()` 对引用类型字段只复制引用，不复制对象本身。要实现深拷贝，需手动克隆每个引用字段，或使用序列化。

详见 [[深拷贝和浅拷贝]]。

### 2.5 toString()

默认实现：

```java
public String toString() {
    return getClass().getName() + "@" + Integer.toHexString(hashCode());
}
// 输出示例：com.example.Person@15db9742
```

> [!tip] 最佳实践：所有自定义类都应重写 toString()，方便日志打印和调试。

```java
@Override
public String toString() {
    return "Person{name='" + name + "', age=" + age + "}";
}
// IDEA 快捷键：Alt + Insert → toString()
```

### 2.6 wait() / notify() / notifyAll()

这三个方法是 Java 多线程中 **等待/通知机制** 的核心，**必须持有对象锁**才能调用。

| 方法 | 作用 |
|------|------|
| `wait()` | 当前线程释放锁，进入 WAITING 状态，等待被唤醒 |
| `wait(long timeout)` | 等待指定毫秒，超时自动唤醒 |
| `notify()` | 随机唤醒一个等待该锁的线程 |
| `notifyAll()` | 唤醒所有等待该锁的线程 |

```java
synchronized (obj) {
    while (condition) {         // 必须用 while 而非 if，防止虚假唤醒
        obj.wait();
    }
    // 执行业务逻辑...
}

synchronized (obj) {
    obj.notify();               // 通知等待线程（必须在 synchronized 块中）
}
```

> [!danger] `wait()/notify()` 必须在 `synchronized` 块中调用，否则抛 `IllegalMonitorStateException`。

> [!tip] **为什么 wait 定义在 Object 而不是 Thread 中？**
> 因为锁是对象级别的，不是线程级别的。任何一个 Java 对象都可以作为锁，所以 wait/notify 必须属于所有对象。

### 2.7 finalize()

```java
@Deprecated(since = "9")
protected void finalize() throws Throwable {
    // GC 回收前调用，但不保证执行时机
    // 不推荐使用：执行不确定、性能差、可能导致对象复活
}
```

> [!warning] finalize() 已被 JDK9 标记为 `@Deprecated`。替代方案：
> - 资源释放用 `try-with-resources` + `AutoCloseable`
> - 内存特殊清理用 `Cleaner` 或 `PhantomReference`

## 三、面试话术

> "Object 是所有 Java 类的父类，共有 11 个方法。我把它们分成三类记忆：
>
> 第一类是对象比较和哈希，也就是 equals 和 hashCode。equals 默认是 == 比地址，String 等类重写后比内容；hashCode 用于哈希表快速定位，重写 equals 必须重写 hashCode。
>
> 第二类是线程通信三剑客，wait、notify、notifyAll。它们必须在 synchronized 块中调用，因为它们依赖对象锁。wait 让线程释放锁并等待，notify 随机唤醒一个等待线程。wait 定义在 Object 中是因为任何对象都可以作为锁。
>
> 第三类是工具类方法：getClass 获取运行时类型，clone 复制对象（默认浅拷贝），toString 打印对象信息。finalize 已经废弃了，不建议使用。
>
> toString 在实际开发中最常用，所有自定义类都应该重写 toString，方便打 log 排查问题。"

## 四、记忆口诀

```
Object 十一法记心间，getClass hashCode equals 在前。
clone 浅拷贝实现 Cloneable，toString 打印要重写。
wait notify notifyAll，多线程同步三剑客，必须锁里调。
finalize 已废弃，资源释放用 try-with-resources 替。

核心契约：equals 同则 hashCode 同，wait 必须锁中行。
```