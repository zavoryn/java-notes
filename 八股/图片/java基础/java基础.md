# Java 基础

面试前需要掌握哪些知识？Java 考前大纲！

后台很多同学会问我，面试都会考些什么呀？我时间紧，面试重点有哪些？八股要背到什么程度才能应付面试？算法需要刷哪些题？

今天我来总结一份面试考前大纲，帮助大家消除迷茫~
大家可以对照着自己练习和考察。

## 1. Java 基础

### 集合 ★★★★★
HashMap, ArrayList, LinkedList, HashSet, ConcurrentHashMap

1. 掌握 ArrayList 和 LinkedList 的源码，区别，使用场景
2. 掌握 HashSet 的底层实现，使用场景，优势
3. 掌握 HashMap，ConcurrentHashMap 的底层原理，区别，使用场景，Hash 冲突解决方式等

### 字符串 ★★★★
String, StringBuilder, StringBuffer

1. Java 的 String 底层是用什么实现的？可变还是不可变？（这个百度考过我）
2. String, StringBuilder, StringBuffer 区别，使用场景
3. String 的 equals() 和 == 的区别，hashCode() 重写规则
4. 常见字符串操作方法（如 split、substring、join），这个很多算法题会用到，想不起来就 g 了。

### 泛型 ★★★★

1. 概念，用途（类型安全、避免强制转换）
2. 泛型擦除？（编译期擦除为原生类型，运行时无泛型信息，解决方法）
3. 泛型类、泛型接口、泛型方法的定义与使用
4. 通配符 ?、? extends T、? super T 的区别与使用场景（概率低，选择性学习）
5. 泛型的限制（不能用于基本类型 int, long 等、不能实例化泛型数组等）（概率低，选择性学习）

### 面向对象 ★★★★
封装、继承、多态。下面这些都挺重要的

1. 面向对象三大特性的概念、实现方式
2. 重写与重载的区别（参数列表、返回值、异常、访问修饰符）
3. 抽象类与接口的区别
4. final 关键字的用法（修饰类、方法、变量）
5. static 关键字的用法（静态变量、静态方法、静态代码块、静态内部类）
6. 值传递与引用传递的区别，Java 是哪种？（举例说明）

### 异常 ★★（概率低，选择性学习）
Exception, Error, RuntimeException

1. 异常体系结构（Throwable 子类：Error、Exception 及细分）
2. try-catch-finally 的执行顺序，finally 一定执行吗？（特殊情况）
3. throw 与 throws 的区别
4. 自定义异常的实现与使用场景

### 其他基础 ★★

1. 基本数据类型与包装类的区别，自动装箱与拆箱（原理、空指针问题）
2. 枚举 Enum 的定义、使用场景及底层实现
3. 注解 Annotation 的概念、分类（内置注解、元注解）及自定义注解
4. Java 版本新特性（11, 17, 21）了解
