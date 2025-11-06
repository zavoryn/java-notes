---
tags:
  - 面试
  - Java Agent
  - 字节码增强
  - JVMTI
  - APM
difficulty: 综合
created: 2026-05-13
sources: 10+
---

# Java Agent 面试题大全

> 本文整理自掘金、CSDN、博客园、美团技术博客、腾讯云开发者社区等公开技术文章的面试高频考点。
> 覆盖 Java Agent 原理、JVMTI、字节码增强、APM 工具链等核心知识点。

---

## 一、基础概念篇（⭐ 入门级）

### Q1: 什么是 Java Agent？它的作用是什么？

**答：** Java Agent 是一种特殊的 Java 程序，它不是通过 `main` 方法启动的独立程序，而是必须依附在一个 Java 应用程序（JVM）上，与主程序运行在同一个进程中。

核心作用：
- **无侵入式字节码增强**：在类加载时拦截并修改字节码，不需要修改源代码
- **性能监控（APM）**：如 SkyWalking、Pinpoint 通过 Agent 收集方法耗时、链路追踪
- **热修复**：动态替换已加载类的字节码（如 Arthas 的 redefine 命令）
- **诊断调试**：运行时查看方法调用栈、参数、返回值（如 Arthas）
- **安全审计**：监控敏感 API 调用（如 `System.exit`）
- **Mock 测试**：如 Mockito 的 Agent 模式

> **来源：** [Java Agent 与 JVMTI 深度解析 - 掘金](https://juejin.cn/post/7508919896439095335) | [腾讯云开发者社区](https://cloud.tencent.com/developer/article/2050980)

---

### Q2: Java Agent 有哪两种加载方式？区别是什么？

**答：**

| 对比项 | premain（启动时加载） | agentmain（运行时加载） |
|--------|----------------------|------------------------|
| **引入版本** | JDK 1.5 | JDK 1.6 |
| **加载时机** | JVM 启动时、`main` 方法之前 | JVM 运行期间，通过 Attach API 动态注入 |
| **使用方式** | `-javaagent:agent.jar=options` | `VirtualMachine.attach(pid).loadAgent(jarPath)` |
| **可操作的类** | 所有类（包括未加载的） | 已加载的类（需 retransform） |
| **限制** | 较少限制 | 不能增减父类/接口，新增方法只能是 private static/final，不能修改字段，类访问符不能变化 |
| **典型应用** | SkyWalking、Pinpoint | Arthas |
| **性能影响** | 较小（类加载时处理） | 较大（需在安全点暂停所有线程） |

**面试关键点：** `premain` 方法会在 `main` 方法之前被调用。多个 `-javaagent` 参数按指定顺序执行，但必须放在 `-jar` 参数之前，否则不会生效。

```java
// premain 方法签名
public static void premain(String agentArgs, Instrumentation inst);

// agentmain 方法签名
public static void agentmain(String agentArgs, Instrumentation inst);
```

> **来源：** [阿里云开发者社区](https://developer.aliyun.com/article/834736) | [博客园 - Java探针](https://www.cnblogs.com/aspirant/p/8796974.html) | [深入理解 Skywalking Agent](https://www.cnblogs.com/stateis0/p/16099932.html)

---

### Q3: Java Agent 的 MANIFEST.MF 配置项有哪些？

**答：**

```
Manifest-Version: 1.0
Premain-Class: com.example.MyAgent          # premain 方法所在类
Agent-Class: com.example.MyAgent            # agentmain 方法所在类
Can-Redefine-Classes: true                   # 是否支持类重定义
Can-Retransform-Classes: true                # 是否支持类重转换
Boot-Class-Path: javassist.jar               # Agent 依赖加入 Bootstrap ClassLoader
```

**面试陷阱：** MANIFEST.MF 文件最后一行必须是空行，冒号后面必须有一个空格。

> **来源：** [博客园 - Java探针](https://www.cnblogs.com/aspirant/p/8796974.html)

---

### Q4: 什么是 Instrumentation API？核心方法有哪些？

**答：** `java.lang.instrument.Instrumentation` 是 Java Agent 技术的核心接口，提供了对 JVM 底层操作的抽象。

**核心方法：**

| 方法 | 作用 |
|------|------|
| `addTransformer(ClassFileTransformer)` | 注册类文件转换器，按注册顺序依次调用 |
| `addTransformer(transformer, canRetransform)` | 注册转换器并指定是否可用于重转换 |
| `retransformClasses(Class<?>...)` | 重新转换已加载的类（触发 ClassFileTransformer） |
| `redefineClasses(ClassDefinition...)` | 直接替换类的字节码（完全替换） |
| `getAllLoadedClasses()` | 获取 JVM 中所有已加载的类 |
| `getInitiatedClasses(ClassLoader)` | 获取指定类加载器加载的类 |
| `appendToBootstrapClassLoaderSearch(JarFile)` | 将 JAR 加入 Bootstrap ClassLoader 搜索路径 |
| `appendToSystemClassLoaderSearch(JarFile)` | 将 JAR 加入 System ClassLoader 搜索路径 |
| `isModifiableClass(Class<?>)` | 检查类是否可被修改 |
| `isRetransformClassesSupported()` | 检查是否支持重转换 |
| `isRedefineClassesSupported()` | 检查是否支持重定义 |

> **来源：** [掘金 - Java Agent与JVMTI深度解析](https://juejin.cn/post/7508919896439095335)

---

### Q5: retransformClasses 和 redefineClasses 的区别？

**答：**

| 对比项 | retransformClasses | redefineClasses |
|--------|-------------------|-----------------|
| **引入版本** | JDK 1.6 | JDK 1.5 |
| **工作方式** | 基于原始字节码重新运行 Transformer 链 | 直接替换整个类定义 |
| **是否可恢复** | 是，保留被修改的内容 | 否，完全覆盖，不可恢复 |
| **限制** | 较少 | 不能修改/删除 field 和 method，包括方法参数、名称和返回值 |
| **推荐** | ✅ 优先使用 | ❌ 有较多限制 |

**面试回答要点：** Redefine 是 Java 1.5 引入的，有很多缺陷（完全替换后不可恢复，不能修改方法结构）。Retransform 是 Java 1.6 引入的，解决了 Redefine 的大部分问题，允许增量修改。

> **来源：** [深入理解 Skywalking Agent - 博客园](https://www.cnblogs.com/stateis0/p/16099932.html) | [Arthas 作者文章](https://hengyun.tech/arthas-jad/)

---

## 二、JVMTI 篇（⭐⭐ 中级）

### Q6: 什么是 JVMTI？它与 Java Agent 的关系？

**答：** JVMTI（JVM Tool Interface）是 JVM 提供的原生编程接口（Native Interface），用于开发调试、监控、性能分析等底层工具。

**层级关系：**
```
Java Agent (应用层)
    ↓ 使用
Instrumentation API (Java 层接口)
    ↓ 底层依赖
JVMTI (JVM 原生 C/C++ 接口)
```

Java Agent 通过 Instrumentation API 与 JVMTI 交互。例如 `Instrumentation#addTransformer` 方法最终通过 JVMTI 的 `AddToBootstrapClassLoaderSearch` 和类转换事件实现字节码修改。

> **来源：** [掘金 - Java Agent与JVMTI深度解析](https://juejin.cn/post/7508919896439095335)

---

### Q7: JVMTI 的核心功能有哪些？

**答：**

**1. 事件通知（回调机制）：**
- `ClassPrepare`：类准备完成
- `MethodEntry / MethodExit`：方法进入/退出
- `GarbageCollectionStart / Finish`：GC 事件
- `ClassFileLoadHook`：类文件加载钩子

**2. 功能函数：**
- `GetLoadedClasses`：获取所有已加载的类
- `Allocate / Deallocate`：直接操作 JVM 内存
- `SuspendThread / ResumeThread`：控制线程状态
- `GetStackTrace`：获取调用栈

**3. 典型工具的 JVMTI 应用：**
- **JDWP 调试协议**：基于 JVMTI 实现断点、单步执行
- **Arthas watch 命令**：利用 MethodEntry/MethodExit 事件监控方法调用
- **Arthas stack 命令**：通过 GetStackTrace 获取调用栈
- **Async Profiler**：通过 JVMTI 映射 Java 方法符号，结合 perf_events 获取 CPU 周期事件

> **来源：** [掘金 - Java Agent与JVMTI深度解析](https://juejin.cn/post/7508919896439095335) | [GeeksforGeeks](https://www.geeksforgeeks.org/java/introduction-to-java-agent-programming/)

---

### Q8: ClassFileTransformer 的执行顺序是什么？

**答：**

- 可以注册多个 `ClassFileTransformer`，**按注册顺序依次调用**
- 每个 Transformer 接收上一个 Transformer 处理后的字节码
- 如果某个 Transformer 返回 `null`，表示不修改（传递原始字节码给下一个）
- 如果某个 Transformer 返回非 null 的 byte 数组，则后续 Transformer 接收到的是修改后的字节码

```java
public interface ClassFileTransformer {
    byte[] transform(ClassLoader         loader,
                     String              className,
                     Class<?>            classBeingRedefined,
                     ProtectionDomain    protectionDomain,
                     byte[]              classfileBuffer) throws IllegalClassFormatException;
}
```

> **来源：** [腾讯云 - Java Agent 入门](https://cloud.tencent.com/developer/article/1893784) | [阿里云开发者社区](https://developer.aliyun.com/article/834736)

---

## 三、字节码操作框架篇（⭐⭐ 中级）

### Q9: 常见的字节码操作框架有哪些？对比一下？

**答：**

| 框架 | 层级 | 性能 | 易用性 | 典型用户 |
|------|------|------|--------|----------|
| **ASM** | 底层 | 最高（直接操作字节码） | 最低（需要了解 JVM 字节码指令） | SkyWalking、Pinpoint、JaCoCo |
| **Javassist** | 中层 | 中等 | 高（使用 Java 语法编写方法体） | Arthas（部分） |
| **Byte Buddy** | 高层 | 中等（基于 ASM） | 最高（流式 API） | SkyWalking（新版）、Mockito |

**面试进阶：** 为什么 SkyWalking 从 Javassist 迁移到 Byte Buddy？
- Byte Buddy 基于 ASM，性能更优
- API 更简洁，降低插件开发门槛
- 更好的类加载器隔离支持

> **来源：** [美团技术团队 - 字节码增强技术探索](https://tech.meituan.com/2019/09/05/java-bytecode-enhancement.html) | [CSDN - javaagent + ASM](https://blog.csdn.net/Tanganling/article/details/132858851)

---

### Q10: 用 Javassist 实现方法耗时统计的 Agent 的核心代码思路？

**答：**

核心思路：在 `transform` 方法中，使用 Javassist 修改目标方法的字节码，在方法前后添加计时逻辑。

```java
// 1. 实现 ClassFileTransformer
public class MyTransformer implements ClassFileTransformer {
    @Override
    public byte[] transform(ClassLoader loader, String className, ...,
                            byte[] classfileBuffer) {
        if (methodMap.containsKey(className)) {
            CtClass ctClass = ClassPool.getDefault().get(className);
            for (String methodName : methodMap.get(className)) {
                CtMethod oldMethod = ctClass.getDeclaredMethod(methodName);
                // 2. 原方法改名
                oldMethod.setName(methodName + "$old");
                // 3. 复制原方法创建新方法
                CtMethod newMethod = CtNewMethod.copy(oldMethod, methodName, ctClass, null);
                // 4. 设置新方法体：计时 + 调用原方法
                newMethod.setBody("{long s=System.currentTimeMillis();"
                    + methodName + "$old($$);"
                    + "System.out.println(\"cost:\"+(System.currentTimeMillis()-s));}");
                ctClass.addMethod(newMethod);
            }
            return ctClass.toBytecode();
        }
        return null;
    }
}
```

> **来源：** [博客园 - Java探针](https://www.cnblogs.com/aspirant/p/8796974.html)

---

## 四、APM 工具原理篇（⭐⭐⭐ 高级）

### Q11: SkyWalking Agent 的架构设计是怎样的？

**答：**

SkyWalking Java Agent 采用**微内核 + 插件化**架构：

```
apm-agent-core (核心)
├── 启动、加载配置
├── 加载插件、修改字节码
├── 记录调用数据
└── 发送到后端 OAP
apm-sdk-plugin (插件)
├── Jedis 插件
├── Dubbo 插件
├── RocketMQ 插件
├── Kafka 插件
└── ... 各种中间件插件
```

**核心设计要点：**
1. 使用 `premain` 方式挂载，使用 Byte Buddy（基于 ASM）实现字节码插桩
2. 入口类 `SkyWalkingAgent#premain`
3. 插件通过 `skywalking-plugin.def` 文件声明，自动发现加载
4. 每个 ClassLoader 实例对应一个 `AgentClassLoader`，解决类加载隔离问题

> **来源：** [深入理解 Skywalking Agent - 博客园](https://www.cnblogs.com/stateis0/p/16099932.html)

---

### Q12: SkyWalking 中 Trace / Segment / Span 的关系？

**答：**

```
Trace (全局链路)
 ├── Segment (JVM A 中的一次调用)
 │    ├── Span 1 (Tomcat 入口 Span)
 │    ├── Span 2 (Service 方法 Span)
 │    └── Span 3 (Jedis 出口 Span)
 ├── Segment (JVM B 中的一次调用)
 │    ├── Span 4 (Dubbo 入口 Span)
 │    └── Span 5 (MySQL 出口 Span)
 └── ...
```

- **Trace**：一次完整调用链路，有全局唯一 ID
- **Segment**：一个 JVM 中一个线程的一次调用链路（多个 Span 组成）
- **Span**：一个组件的调用信息，是 Trace 中的一个节点

SkyWalking 使用**栈结构**管理 Span 生命周期：
- Span 创建 → 入栈
- Span 结束 → 出栈
- 栈为空 → Segment 结束，发送到 OAP Server

> **来源：** [深入理解 Skywalking Agent - 博客园](https://www.cnblogs.com/stateis0/p/16099932.html)

---

### Q13: SkyWalking 如何处理异步 Trace？

**答：**

SkyWalking 使用 `ThreadLocal` 存储当前线程的 Span 信息。对于异步场景，提供两种机制：

**1. 跨线程链接（capture / continued）：**
- `capture()`：将当前栈顶 Span 复制为快照
- `continued(snapshot)`：在子线程中将快照恢复为父 Span
- 适用场景：主线程 → 子线程的 Span 链接

**2. 跨线程 Span（prepareForAsync / asyncFinish）：**
- `prepareForAsync()`：标记 Span 进入异步模式（在 A 线程）
- `asyncFinish()`：标记 Span 结束（在 B 线程）
- 适用场景：一个 Span 跨越两个线程（如异步 HttpClient）

**注意事项：**
- `prepareForAsync` 后必须调用 `stopSpan`
- `asyncFinish` 必须在其他线程被调用，否则 Segment 不会结束
- 必须正确调用 `ContextManager.stopSpan()`，否则会导致内存泄漏

> **来源：** [深入理解 Skywalking Agent - 博客园](https://www.cnblogs.com/stateis0/p/16099932.html)

---

### Q14: Arthas 如何实现运行时诊断？

**答：**

Arthas 通过 `agentmain` 动态附加到目标 JVM：

```java
// Arthas AgentBootstrap 核心
public static void agentmain(String args, Instrumentation inst) {
    // 1. 使用自定义 ArthasClassLoader 加载核心类（类加载隔离）
    ClassLoader loader = getClassLoader(inst, arthasCoreJar);
    Class<?> bootstrapClass = loader.loadClass("com.taobao.arthas.core.server.ArthasBootstrap");
    // 2. 获取 ArthasBootstrap 单例
    Object bootstrap = bootstrapClass.getMethod("getInstance", Instrumentation.class, String.class)
                                     .invoke(null, inst, args);
}
```

**关键设计：**
- 使用自定义 `ArthasClassLoader` 避免与应用类冲突
- 利用 Instrumentation API 实现类重定义（watch、tt 等命令）
- 通过字节码增强在目标方法前后插入监控代码

**Arthas 常用命令对应的 JVMTI 能力：**
| 命令 | 底层能力 |
|------|----------|
| watch | MethodEntry/MethodExit 事件监控 |
| stack | GetStackTrace 获取调用栈 |
| tt（Time Tunnel） | 断点 + 单步执行 |
| redefine | Instrumentation.redefineClasses |
| retransform | Instrumentation.retransformClasses |

> **来源：** [掘金 - Java Agent与JVMTI深度解析](https://juejin.cn/post/7508919896439095335)

---

### Q15: Async Profiler 的实现原理？

**答：**

Async Profiler 架构：

```
1. 结合 perf_events 获取 CPU 周期事件
2. 低开销采样技术（AsyncGetCallTrace）
   - 无需暂停 JVM 进程
   - 直接访问 JVM 内部栈结构
   - 采样间隔达到 1ms 级别
3. 使用 JVMTI 映射 Java 方法符号
4. 混合栈合并算法处理 JNI 调用
```

**关键点：** AsyncGetCallTrace 是 JVM 内部的一个非标准 API，不需要在安全点暂停线程，因此开销极低。它直接读取 JVM 内部的栈结构，避免了 safepoint bias 问题。

> **来源：** [掘金 - Java Agent与JVMTI深度解析](https://juejin.cn/post/7508919896439095335)

---

## 五、实战与深度篇（⭐⭐⭐ 高级）

### Q16: 多个 Java Agent 同时使用时为什么会冲突？如何解决？

**答：**

**冲突原因：**
- 多个 Agent 注册的 `ClassFileTransformer` 按顺序链式调用
- 后一个 Agent 看到的是前一个 Agent 修改后的字节码
- 如果两个 Agent 对同一个类做增强，可能导致：
  - 字节码结构被破坏
  - 增强逻辑互相覆盖
  - 类加载失败

**解决方案：**
1. 确保 Agent 的增强逻辑幂等（检查是否已被增强）
2. 使用 NopTransformer 模式：只修改需要的方法，其他返回 null
3. 控制加载顺序：`-javaagent` 参数的顺序决定了 Transformer 的执行顺序
4. 使用 Byte Buddy 的 `AgentBuilder` 的 avoidance 机制避免冲突

> **来源：** [多个 Java Agent 同时使用的类增强冲突问题及分析 - 博客园](https://www.cnblogs.com/huaweiyun/p/16876537.html)

---

### Q17: 能否对 JDK 核心类进行插桩？有哪些限制？

**答：**

**可以，但有严格限制：**
- JDK 核心类由 Bootstrap ClassLoader 加载
- Agent 需要通过 `appendToBootstrapClassLoaderSearch` 将自己的 JAR 加入 Bootstrap ClassLoader 搜索路径
- 不能对 `java.lang.Object`、`java.lang.String` 等极其基础的类进行插桩（会导致循环依赖）
- 插桩逻辑不能引用不在 Bootstrap ClassLoader 路径中的类

**实践建议：** 如果需要监控 JDK 类的方法调用，优先考虑使用 JVMTI 的 MethodEntry/MethodExit 事件而非字节码插桩。

> **来源：** [关于 Java Agent 的使用、工作原理及 hotspot 源码解析 - 掘金](https://juejin.cn/post/7361564628952399899)

---

### Q18: SkyWalking 的插件类加载隔离设计为什么这样做？

**答：**

SkyWalking 为每个业务 ClassLoader 创建一个对应的 `AgentClassLoader`，parent 设为业务 ClassLoader。

**为什么不用默认的 AgentClassLoader（parent 是 AppClassLoader）？**

假设 Jedis 是由自定义 ClassLoader 加载的：
- 用默认 AgentClassLoader → 向上查找到 AppClassLoader → 找不到 Jedis 类 → 失败
- 用新的 AgentClassLoader（parent = Jedis 的 ClassLoader）→ 可以正确访问 Jedis 类 → 成功

**核心原理：** 解决不同 ClassLoader 命名空间隔离导致的类访问问题。

> **来源：** [深入理解 Skywalking Agent - 博客园](https://www.cnblogs.com/stateis0/p/16099932.html)

---

### Q19: Attach API 的工作原理是什么？

**答：**

Attach API 是 JVM 进程之间的通信桥梁，底层通过 Socket 通信：

```
JVM A (Attach 端)                          JVM B (目标进程)
    |                                           |
    | VirtualMachine.attach(pid)                |
    | ────── Socket 连接 ──────────────────→    |
    |                                           |
    | loadAgent(agentJarPath)                   |
    | ────── 发送加载指令 ─────────────────→    |
    |                                           | 触发 agentmain 方法
    |                                           | 注册 Transformer
    |                                           | retransformClasses
    |                                           |
```

常用工具的 Attach API 使用：
- `jstack`：通过 Attach API 获取线程栈
- `jcmd`：通过 Attach API 发送诊断命令
- `jps`：通过 Attach API 列出 Java 进程

> **来源：** [腾讯云 - 1000个字搞懂JavaAgent](https://cloud.tencent.com/developer/article/2050980)

---

### Q20: Java Agent 与 AOP 的区别？

**答：**

| 对比项 | Java Agent | AOP（Spring AOP） |
|--------|-----------|-------------------|
| **侵入性** | 无侵入，不需要修改任何源代码 | 需要框架支持（Spring 容器管理） |
| **作用范围** | 所有 JVM 中的类 | 仅 Spring 管理的 Bean |
| **实现层级** | JVM 级别，修改字节码 | 框架级别，动态代理/CGLIB |
| **生效时机** | 类加载时或运行时 | 运行时 |
| **配置方式** | `-javaagent` 参数或 Attach API | 注解/XML 配置 |
| **适用场景** | APM、全链路追踪、热修复 | 业务逻辑横切关注点 |
| **局限性** | 开发复杂，需要了解字节码 | 仅限 Spring 生态 |

> **来源：** [CSDN - Java Agent与Instrumentation](https://blog.csdn.net/qq_43414012/article/details/149905354)

---

## 六、场景设计题（⭐⭐⭐ 高级）

### Q21: 如何设计一个无侵入的方法耗时统计 Agent？

**设计思路：**

1. **Agent 入口**：实现 `premain` 方法，注册 `ClassFileTransformer`
2. **类匹配**：通过配置文件或注解指定需要监控的类/方法
3. **字节码增强**：使用 Byte Buddy 或 ASM 在目标方法前后插入计时代码
4. **数据收集**：将耗时数据通过内存队列异步上报
5. **数据展示**：提供 HTTP 接口或输出到文件

```java
public class TimingAgent {
    public static void premain(String args, Instrumentation inst) {
        new AgentBuilder.Default()
            .type(ElementMatchers.nameContains("Service"))
            .transform((builder, typeDescription, classLoader, module) ->
                builder.method(ElementMatchers.any())
                       .intercept(Advice.to(TimingAdvice.class))
            ).installOn(inst);
    }

    public static class TimingAdvice {
        @Advice.OnMethodEnter
        public static long onEnter() {
            return System.currentTimeMillis();
        }

        @Advice.OnMethodExit(onThrowable = Throwable.class)
        public static void onExit(@Advice.Enter long start,
                                   @Advice.Origin String method) {
            System.out.println(method + " cost: " + (System.currentTimeMillis() - start) + "ms");
        }
    }
}
```

> **来源：** [美团技术团队 - 字节码增强技术探索](https://tech.meituan.com/2019/09/05/java-bytecode-enhancement.html) | [Byte Buddy 官方文档](https://bytebuddy.net/)

---

### Q22: 如何基于 SkyWalking Agent 实现全链路压测？

**答：**

**核心问题：** 压测过程中不能产生脏数据，影子流量不能进入正式数据库。

**设计方案：**
1. 利用 SkyWalking Agent 的插件机制，对需要隔离的组件（DB、MQ、Redis）编写压测增强插件
2. 在 SQL 执行时，判断当前是否为影子流量（通过 ThreadLocal 传递标记）
3. 如果是影子流量，切换到影子数据源
4. 使用过滤器模式将"压测增强"和"全链路 Trace 增强"串联

**关键实现：**
- 对同一个类实现多个插件，包装成过滤器链
- 影子流量标记通过 SkyWalking 的 Span Tag 传递
- 使用 SkyWalking 的跨线程传播机制传递压测标记

> **来源：** [深入理解 Skywalking Agent - 博客园](https://www.cnblogs.com/stateis0/p/16099932.html)

---

### Q23: 如何防止 Agent 插件内存泄漏？

**答：**

**常见泄漏场景：**
1. `ContextManager.stopSpan()` 未正确调用 → ThreadLocal 不清除 → Segment 累积
2. 异步 Span 未调用 `asyncFinish` → Segment 不会结束
3. `prepareForAsync` 后忘记 `stopSpan`
4. Segment 的 refs 链表无限增长

**防范措施：**
1. 确保每个 `before` 都有对应的 `after`，使用 try-finally 模式
2. `capture/continued` 时确保栈顶有 Span
3. 限制 Segment 的 Span 数量上限
4. 使用 WeakReference 存储 Span 相关对象
5. 添加超时兜底机制：Segment 超过一定时间自动结束

> **来源：** [深入理解 Skywalking Agent - 博客园](https://www.cnblogs.com/stateis0/p/16099932.html)

---

## 七、综合面试真题（大厂高频）

### Q24: [阿里真题] 类加载时能否修改字节码？如何实现？

**答：** 可以。使用 Java Agent 技术实现。核心流程：
1. 实现 `premain` 方法获取 `Instrumentation` 实例
2. 调用 `addTransformer` 注册 `ClassFileTransformer`
3. 在 `transform` 方法中使用 ASM/Javassist/Byte Buddy 修改字节码
4. 返回修改后的 byte 数组

这也是类加载机制（双亲委派模型）的一个重要延伸应用。

> **来源：** [博客园 - Java探针（阿里面试题）](https://www.cnblogs.com/aspirant/p/8796974.html)

---

### Q25: [字节跳动] premain 和 agentmain 能否在同一个 Agent 中同时使用？

**答：** 可以。在 MANIFEST.MF 中同时配置 `Premain-Class` 和 `Agent-Class` 指向同一个类，该类同时实现 `premain` 和 `agentmain` 方法。Arthas 就是这种模式。

但需要注意：
- 两个方法中的 `Instrumentation` 实例可能不同
- `premain` 获取的 Instrumentation 默认支持 retransform，`agentmain` 需要显式配置
- 两种模式对类的修改能力不同

> **来源：** [掘金 - Java Agent与JVMTI深度解析](https://juejin.cn/post/7508919896439095335)

---

### Q26: [美团] 字节码增强在美团的应用场景？

**答：** 美团技术团队在以下场景使用了字节码增强：
1. **服务治理**：无侵入式链路追踪
2. **性能监控**：方法级耗时统计
3. **故障排查**：线上问题快速定位
4. **全链路压测**：影子库流量隔离

技术选型：ASM（高性能场景） + Byte Buddy（通用场景）

> **来源：** [美团技术团队 - 字节码增强技术探索](https://tech.meituan.com/2019/09/05/java-bytecode-enhancement.html)

---

### Q27: agentmain 加载时的性能影响有多大？为什么？

**答：** agentmain 的性能影响主要来自：
1. **安全点暂停**：JVM 需要在安全点暂停所有线程，然后触发 ClassFileLoadHook 事件修改正在运行的字节码
2. **类重转换开销**：`retransformClasses` 需要重新运行 Transformer 链
3. **JIT 退化**：修改字节码后，JIT 编译的机器码失效，需要重新解释执行和编译

因此，SkyWalking 等性能敏感的 APM 工具选择 `premain` 模式（类加载时一次性处理），而不是 `agentmain` 模式。

> **来源：** [深入理解 Skywalking Agent - 博客园](https://www.cnblogs.com/stateis0/p/16099932.html) | [Develotters](https://develotters.com/posts/should-you-use-java-agents-to-instrument-your-application/)

---

### Q28: 如何实现 Agent 的热升级？

**答：**

1. **agentmain + retransform 模式**：通过 Attach API 加载新版本 Agent，调用 `retransformClasses` 重新增强已加载的类
2. **双 Agent 模式**：先通过 agentmain 卸载旧 Agent 的 Transformer（`removeTransformer`），再加载新 Agent
3. **注意点**：
   - `removeTransformer` 只阻止后续的类加载触发该 Transformer，不影响已增强的类
   - 要恢复已增强的类，必须调用 `retransformClasses` 重新走一遍剩余的 Transformer 链

> **来源：** [CSDN - Java Agent与Instrumentation](https://blog.csdn.net/qq_43414012/article/details/149905354)

---

## 八、知识图谱总结

```
Java Agent 面试知识体系
├── 基础概念
│   ├── premain vs agentmain
│   ├── Instrumentation API
│   ├── MANIFEST.MF 配置
│   └── ClassFileTransformer
├── 底层原理
│   ├── JVMTI（事件回调 + 功能函数）
│   ├── Attach API（进程间 Socket 通信）
│   └── 字节码增强链（Transformer 链式调用）
├── 字节码框架
│   ├── ASM（底层高性能）
│   ├── Javassist（中层易用）
│   └── Byte Buddy（高层抽象）
├── APM 工具原理
│   ├── SkyWalking（premain + 插件化）
│   ├── Arthas（agentmain + 动态诊断）
│   ├── Pinpoint（字节码插桩 + 链路追踪）
│   └── Async Profiler（JVMTI + perf_events）
├── 核心模型
│   ├── Trace / Segment / Span
│   ├── 异步 Trace（capture/continued + prepareForAsync/asyncFinish）
│   └── 类加载隔离（AgentClassLoader）
└── 实战问题
    ├── 多 Agent 冲突
    ├── 内存泄漏防范
    ├── 热升级机制
    └── 全链路压测
```

---

## 参考来源

1. [Java Agent 与 JVMTI 深度解析 - 掘金](https://juejin.cn/post/7508919896439095335)
2. [字节码增强技术探索 - 美团技术团队](https://tech.meituan.com/2019/09/05/java-bytecode-enhancement.html)
3. [字节码增强技术之 Java Agent 入门 - 阿里云](https://developer.aliyun.com/article/834736)
4. [Java Agent 与字节码注入 - 极客时间](https://time.geekbang.org/column/article/41186)
5. [1000个字搞懂JavaAgent技术 - 腾讯云](https://cloud.tencent.com/developer/article/2050980)
6. [深入理解 Skywalking Agent - 博客园](https://www.cnblogs.com/stateis0/p/16099932.html)
7. [Java探针技术 - 阿里面试题 - 博客园](https://www.cnblogs.com/aspirant/p/8796974.html)
8. [多个 Java Agent 同时使用的类增强冲突问题 - 博客园](https://www.cnblogs.com/huaweiyun/p/16876537.html)
9. [Java Agent 工作原理及 hotspot 源码解析 - 掘金](https://juejin.cn/post/7361564628952399899)
10. [Introduction to Java Agent Programming - GeeksforGeeks](https://www.geeksforgeeks.org/java/introduction-to-java-agent-programming/)
