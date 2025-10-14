---
tags: [八股, java基础, SPI, JDBC, 设计模式]
status: 已完成
---

# SPI 机制

> [!summary] 我的速记
> **SPI = 服务发现机制**，接口在调用方，实现在第三方 jar。通过 `META-INF/services/接口全限定名` 文件声明实现类，`ServiceLoader.load()` 懒加载。
> **经典案例：** JDBC 驱动加载（无需知道是 MySQL 还是 Oracle）、SLF4J 日志门面。
> **缺点：** 一次性加载所有实现。**Dubbo SPI** 改进了：支持按名称按需加载。

## 1. SPI 是什么？

> **SPI（Service Provider Interface）** 是 JDK 内置的**服务发现机制**。

核心思想：**接口定义在调用方，实现在提供方。** 通过配置文件动态加载接口的实现类，实现**解耦**和**可插拔**。

```
调用方（定义接口） ⇔ SPI 配置文件 ⇔ 提供方（实现接口）
   ↑                                  ↑
   知道接口，不知道具体实现           提供具体实现类
```

> **SPI vs API**：API 是接口定义+实现在一起，调用方直接调用；SPI 是接口定义在调用方，实现在第三方扩展包中，由 SPI 机制动态发现并加载。

## 2. SPI 的实现方式

### 配置文件位置

```
META-INF/services/接口全限定名
```

文件内容是**实现类的全限定名**，每行一个。

### 示例

```java
// 1. 定义接口（通常在核心模块）
package com.example.spi;

public interface PaymentService {
    void pay(int amount);
}

// 2. 实现类（在扩展模块）
package com.example.spi.alipay;

public class AlipayService implements PaymentService {
    @Override
    public void pay(int amount) {
        System.out.println("支付宝支付: " + amount + " 元");
    }
}
```

**META-INF/services/com.example.spi.PaymentService**：

```
com.example.spi.alipay.AlipayService
com.example.spi.wechat.WechatPayService
```

## 3. ServiceLoader 加载机制

```java
// 加载所有 PaymentService 的实现
ServiceLoader<PaymentService> loader = ServiceLoader.load(PaymentService.class);
for (PaymentService service : loader) {
    service.pay(100);
}
// 输出：
// 支付宝支付: 100 元
// 微信支付: 100 元
```

### ServiceLoader 加载原理

```
ServiceLoader.load()
    → 读取 META-INF/services/接口名 文件
    → 解析每一行的实现类全限定名
    → 通过 Class.forName() 加载实现类
    → 调用 newInstance() 创建实例
    → 懒加载：只有在遍历时才真正加载
```

> ServiceLoader 是**延迟加载**（Lazy Loading）的，只有在遍历迭代器时才真正加载和实例化。

## 4. SPI 的应用场景

| 场景 | 说明 | 配置文件位置 |
|------|------|-------------|
| **JDBC 驱动加载** | 连接数据库时自动加载对应驱动 | `META-INF/services/java.sql.Driver` |
| **SLF4J 日志门面** | slf4j 是接口，logback/log4j2 是实现 | `META-INF/services/org.slf4j.spi.SLF4JServiceProvider` |
| **Dubbo 扩展点加载** | Dubbo 自定义了 SPI，支持按需加载 | `META-INF/dubbo/` 目录 |
| **Spring Boot 自动配置** | `spring.factories` 文件，思想类似 SPI | `META-INF/spring.factories` |
| **Spring SPI** | `SpringFactoriesLoader` | `META-INF/spring.factories` |

### JDBC 中 SPI 的工作原理

```java
// 只需要几行代码，不需要知道具体驱动
Connection conn = DriverManager.getConnection(
    "jdbc:mysql://localhost:3306/test", "root", "password");

// 底层原理：
// 1. DriverManager 初始化时调用 ServiceLoader.load(Driver.class)
// 2. 加载 META-INF/services/java.sql.Driver 中的实现类
// 3. MySQL 驱动包中该文件内容：com.mysql.cj.jdbc.Driver
```

## 5. SPI 的优缺点

### 优点

- **解耦**：调用方不需要硬编码实现类，只需知道接口
- **可扩展**：新增实现只需添加 jar + 配置文件，无需改代码
- **插件化**：实现模块的可插拔，符合开闭原则

### 缺点

- **加载所有实现**：`ServiceLoader` 无法按需加载，会实例化所有实现类（即使不需要）
- **遍历才能获取**：ServiceLoader 不支持通过某个 key 直接获取指定实现
- **线程安全**：ServiceLoader 迭代器不是线程安全的
- **配置繁琐**：每个提供方都要在 `META-INF/services` 下创建配置文件

### Dubbo SPI 的改进

```java
// JDK SPI: 加载所有实现
ServiceLoader.load(XxxService.class);  // 全量加载

// Dubbo SPI: 按需加载，支持按名称获取
ExtensionLoader<XxxService> loader = ExtensionLoader.getExtensionLoader(XxxService.class);
XxxService service = loader.getExtension("alipay");  // 按 key 获取，不加载全部
```

> Dubbo SPI 是对 JDK SPI 的增强，支持**按名称加载**、**依赖注入（IoC）**、**AOP/Wrapper 机制**、**自适应扩展**等特性。

## 6. Spring Boot 自动配置和 SPI 的关系

Spring Boot 的自动配置本质是利用了类似 SPI 的思想：

```properties
# META-INF/spring.factories（Spring Boot 2.x）
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.MyAutoConfiguration

# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports (Spring Boot 3.x)
```

> Spring 的 `SpringFactoriesLoader` 借鉴了 JDK SPI 的思路，但更强大：支持条件装配（`@ConditionalOnClass` 等）、排序、多文件合并等。

## 7. 面试话术

> "SPI 是 JDK 内置的服务发现机制，它的核心思想是面向接口编程 + 配置文件动态加载。调用方只依赖接口，通过 `META-INF/services/` 目录下的配置文件声明实现类，由 `ServiceLoader` 动态加载。最经典的例子就是 JDBC：我们写代码时只需要 `DriverManager.getConnection()`，不需要知道具体是 MySQL 还是 Oracle 驱动，具体的驱动实现由 SPI 机制自动发现和加载。"

> "JDK SPI 的缺点是会一次性加载所有实现类，不能按需加载。Dubbo 对 SPI 做了增强，支持按名称获取指定的实现、IoC 注入和 AOP 增强。Spring Boot 的自动配置也借鉴了 SPI 思想，通过 `spring.factories`（Spring Boot 2.x）或 `AutoConfiguration.imports`（Spring Boot 3.x）来声明自动配置类，配合条件注解实现按需装配。"

> 内存口诀：**"接口定规范，配置存全名，懒加载迭代，解耦又灵活。"**
