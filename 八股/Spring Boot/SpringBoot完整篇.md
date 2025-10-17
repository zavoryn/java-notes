---
tags:
  - 八股
  - Spring Boot
status: 待复习
created: 2026-05-05
---

# Spring Boot 完整篇 —— 面试八股文

---

## 一、Spring Boot vs Spring：从"手动档"到"自动档"

### 1.1 核心比喻

> **Spring 是手动档汽车**——你挂挡、踩离合、控制油门，每一步都需要自己掌控。
> **Spring Boot 是自动档汽车**——你只需要踩油门和刹车，变速箱自动帮你完成换挡。

更精准的比喻：

> **Spring 是毛坯房**——墙体框架给你搭好了（IoC 容器、AOP），但装修（配置数据源、事务、MVC、安全）全部自己来。
> **Spring Boot 是精装修套餐房**——开发商（Spring 团队）根据你的需求（你说"我要 Web 应用"），已经把墙刷好了、家具摆上了、电器配齐了，拎包入住。

### 1.2 五大核心区别

| 维度 | Spring Framework | Spring Boot |
|------|-----------------|-------------|
| **配置方式** | XML / Java Config，大量手动配置 | 自动配置，约定大于配置 |
| **依赖管理** | 手动管理版本，容易冲突 | starter 起步依赖，版本仲裁 |
| **部署方式** | 打 war 包，外部 Tomcat | 打 jar 包，内嵌 Tomcat，`java -jar` 直接跑 |
| **监控运维** | 需要自己集成 | Actuator 开箱即用 |
| **本质定位** | 底层框架（IoC + AOP + 模块） | Spring 的快速集成方案（不是新框架） |

### 1.3 一句话总结

> Spring Boot 不是用来替代 Spring 的，它是**站在 Spring 的肩膀上**，用"约定大于配置"的思想，帮你把 Spring 生态的各种组件快速整合到一起。它本质是一个**脚手架/快速集成方案**。

### 1.4 面试标准回答

> "Spring Boot 是基于 Spring Framework 的快速开发框架，核心价值在于**简化 Spring 应用的搭建和开发过程**。它通过四大核心能力实现这一目标：**自动配置**（根据类路径中的 jar 包自动配置 Spring 组件）、**起步依赖**（Starter，一站式引入相关依赖并解决版本冲突）、**内嵌容器**（Tomcat/Jetty/Undertow，jar 包直接运行）、**Actuator 监控**（生产级端点）。需要强调的是，Spring Boot 不是新框架，它是对 Spring 的封装和增强，底层仍然是 Spring 的 IoC 和 AOP。"

---

## 二、自动配置原理 —— 装修公司的"套餐"机制

### 2.1 核心比喻

> **装修公司套餐**：你去装修公司说"我要一个三室两厅的现代简约风格"，设计师直接给你出方案、配材料、定施工队。你不需要知道每个开关用什么型号、每根水管怎么走。
>
> **Spring Boot 自动配置**：你在 pom.xml 里引入 `spring-boot-starter-web`，Spring Boot 就自动帮你配置了 DispatcherServlet、ViewResolver、HttpMessageConverter、Tomcat 容器等 50+ 个 Bean。你不需要自己写这些配置。

### 2.2 自动配置的工作流

```
@SpringBootApplication
      │
      ├── 1. @SpringBootConfiguration (标记为配置类)
      │
      ├── 2. @EnableAutoConfiguration (开启自动配置，核心!)
      │         │
      │         └── @Import(AutoConfigurationImportSelector.class)
      │                    │
      │                    └── 读取 META-INF/spring/
      │                         org.springframework.boot.autoconfigure
      │                         .AutoConfiguration.imports
      │                         (Spring Boot 3.x 新位置)
      │                         (Spring Boot 2.x: spring.factories)
      │                              │
      │                              └── 加载所有 xxxAutoConfiguration 类
      │                                        │
      │                                        └── 条件注解过滤
      │                                              ├── @ConditionalOnClass (类路径有某个类才生效)
      │                                              ├── @ConditionalOnMissingBean (容器中没有该 Bean 才生效)
      │                                              ├── @ConditionalOnProperty (有某个配置才生效)
      │                                              └── @ConditionalOnWebApplication (Web 应用才生效)
      │
      └── 3. @ComponentScan (扫描当前包及子包的 Bean)
```

### 2.3 关键源码解读

**@SpringBootApplication 注解**：

```java
@SpringBootConfiguration    // 等价于 @Configuration
@EnableAutoConfiguration    // 自动配置的入口
@ComponentScan(             // 组件扫描
    excludeFilters = { @Filter(type = FilterType.CUSTOM,
                                classes = TypeExcludeFilter.class) })
public @interface SpringBootApplication {
}
```

**@EnableAutoConfiguration 注解**：

```java
@AutoConfigurationPackage   // 将主配置类所在包注册为自动配置包
@Import(AutoConfigurationImportSelector.class)  // 关键！导入自动配置选择器
public @interface EnableAutoConfiguration {
}
```

**AutoConfigurationImportSelector** 的核心逻辑：

```java
// 简化的核心流程
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
                                                   AnnotationAttributes attributes) {
    // 从 META-INF/spring/xxx.AutoConfiguration.imports 读取所有自动配置类
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
        EnableAutoConfiguration.class, getBeanClassLoader());
    // Spring Boot 3.x 使用 .imports 文件，2.x 使用 spring.factories
    return configurations;
}

// 然后用条件注解过滤：@ConditionalOnClass、@ConditionalOnMissingBean 等
// 只保留满足条件的自动配置类
```

### 2.4 以 DataSource 自动配置为例

```java
// 来自 spring-boot-autoconfigure 中的 DataSourceAutoConfiguration
@AutoConfiguration
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ DataSourcePoolMetadataProvidersConfiguration.class })
public class DataSourceAutoConfiguration {

    @Configuration
    @ConditionalOnMissingBean(DataSource.class)  // 用户没自定义才自动配
    @ConditionalOnProperty(name = "spring.datasource.type")  // 有配置才配
    static class Generic {
        @Bean
        DataSource dataSource(DataSourceProperties properties) {
            // 根据 spring.datasource.type 创建对应的 DataSource
            return properties.initializeDataSourceBuilder().build();
        }
    }
}
```

**解读流程**：
1. `@ConditionalOnClass({DataSource.class})` → 类路径下有 DataSource 才继续（你引入了 JDBC 依赖才有）
2. `@ConditionalOnMissingBean(DataSource.class)` → 你自己没有定义 DataSource Bean 才用自动配置（尊重用户自定义）
3. `@ConditionalOnProperty(name = "spring.datasource.type")` → 你配了连接池类型才创建

### 2.5 条件注解全家福

| 条件注解 | 作用 | 使用场景 |
|----------|------|---------|
| `@ConditionalOnClass` | 类路径存在指定类时生效 | 有 MySQL 驱动才配 DataSource |
| `@ConditionalOnMissingClass` | 类路径不存在指定类时生效 | 没有某个类时用默认实现 |
| `@ConditionalOnBean` | 容器中存在指定 Bean 时生效 | 有 DataSource 才配 JdbcTemplate |
| `@ConditionalOnMissingBean` | 容器中不存在时生效 | 用户没自定义才用自动配置 |
| `@ConditionalOnProperty` | 配置文件中某属性存在或有特定值时 | `spring.cache.type=redis` 才启用 Redis 缓存 |
| `@ConditionalOnResource` | 类路径存在指定资源时 | 存在 logback.xml 用 Logback |
| `@ConditionalOnWebApplication` | 是 Web 应用时生效 | Web 环境才配 DispatcherServlet |
| `@ConditionalOnExpression` | SpEL 表达式为 true | 复杂条件组合 |

### 2.6 面试标准回答

> "Spring Boot 自动配置的核心通过 `@SpringBootApplication` 注解中的 `@EnableAutoConfiguration` 实现。它使用 `AutoConfigurationImportSelector` 从 `META-INF/spring/xxx.AutoConfiguration.imports` 文件（Spring Boot 2.x 用 spring.factories）中加载所有自动配置类的全限定名，然后通过**条件注解**（如 @ConditionalOnClass、@ConditionalOnMissingBean）进行过滤，只让满足条件的配置类生效。这个机制保证了'按需配置'——只有你引入了相关依赖，对应的自动配置才会生效。同时 `@ConditionalOnMissingBean` 保证了用户自定义配置的优先级高于自动配置。"

---

## 三、起步依赖 Starter —— 依赖管理的"一站式采购"

### 3.1 核心比喻

> **传统 Spring**：你要做一道菜，需要去菜市场一个个摊位买——肉、蔬菜、调料、配料（每个 jar 包），还得自己看哪个牌子好、哪个版本搭（jar 版本兼容）。
>
> **Spring Boot Starter**：你在"盒马"（Starter）买一个"酸菜鱼调料包"（spring-boot-starter-web），里面酸菜、鱼片、调料、辣椒全部配齐，版本也是出厂就搭配好的（版本仲裁）。

### 3.2 Starter 的本质

Starter 是一个 **Maven/Gradle 依赖描述符**——它本身不包含功能代码，只负责**传递依赖管理**。

以 `spring-boot-starter-web` 为例，它帮你引入了：

```
spring-boot-starter-web
  ├── spring-boot-starter (核心 Starter: 自动配置、日志)
  ├── spring-boot-starter-json (Jackson)
  ├── spring-boot-starter-tomcat (内嵌 Tomcat)
  ├── spring-web (Spring MVC 核心)
  └── spring-webmvc (Spring MVC)
```

**你只需要写**：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

一个依赖，50+ 个传递依赖自动引入，且**版本已经在 Spring Boot 父 POM 中仲裁好了**。

### 3.3 版本仲裁机制

```xml
<!-- Spring Boot 的父 POM (spring-boot-dependencies) 定义了所有依赖的版本 -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
</parent>

<!-- 里面用 <dependencyManagement> 统一管理了 200+ 个依赖版本 -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>6.1.0</version>  <!-- Spring Boot 3.2 配套的 Spring 6.1 -->
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.15.3</version>  <!-- 经过测试的兼容版本 -->
        </dependency>
        <!-- ... 200+ 个依赖版本统一管理 ... -->
    </dependencies>
</dependencyManagement>
```

### 3.4 常见 Starter 一览

| Starter | 功能 | 引入的核心依赖 |
|---------|------|--------------|
| `spring-boot-starter` | 核心 Starter | 自动配置、日志(logback)、yaml支持 |
| `spring-boot-starter-web` | Web 应用 | Spring MVC、内嵌 Tomcat、Jackson |
| `spring-boot-starter-data-redis` | Redis | Lettuce 客户端、Spring Data Redis |
| `spring-boot-starter-data-jpa` | JPA | Hibernate、Spring Data JPA、连接池 |
| `spring-boot-starter-test` | 测试 | JUnit5、Mockito、AssertJ |
| `spring-boot-starter-security` | 安全 | Spring Security |
| `spring-boot-starter-actuator` | 监控 | Actuator 端点 |
| `spring-boot-starter-aop` | AOP | spring-aop、AspectJ |
| `spring-boot-starter-amqp` | 消息队列 | spring-rabbit (RabbitMQ) |
| `mybatis-spring-boot-starter` | MyBatis | MyBatis、MyBatis-Spring（第三方） |

### 3.5 自定义 Starter 命名规范

- **官方 Starter**：`spring-boot-starter-xxx`（如 spring-boot-starter-web）
- **第三方 Starter**：`xxx-spring-boot-starter`（如 mybatis-spring-boot-starter）

### 3.6 面试标准回答

> "Starter 是 Spring Boot 的依赖管理方案，本质是一个 Maven 的 pom 描述符，不包含业务代码。它的核心价值是**一站式依赖引入 + 版本仲裁**。你只需引入一个 Starter，它就通过传递依赖机制把所有需要的 jar 包都带进来，并且版本已经在 `spring-boot-dependencies` 中经过了充分的兼容性测试。这解决了传统 Spring 项目中最让人头疼的'jar 包版本冲突'问题。"

---

## 四、内嵌 Tomcat —— "自带发电机的房车"

### 4.1 核心比喻

> **传统 Spring 部署**：你的应用是一辆**拖挂式房车**（war 包），需要一台卡车（外部 Tomcat 服务器）才能拖着你跑。你得自己找卡车、挂上去、发动卡车。
>
> **Spring Boot 部署**：你的应用是一辆**自行式房车**（jar 包），自带发动机（内嵌 Tomcat），拧钥匙就能走，想去哪去哪。

### 4.2 内嵌容器的本质

Spring Boot 把 Tomcat 作为普通依赖引入项目中，在启动时通过 Java 代码**编程式地**创建并启动 Tomcat 实例，而不是把应用部署到已经运行的 Tomcat 容器中。

```java
// 简化原理：Spring Boot 内部做的事情
public class TomcatEmbeddedServletContainer {
    Tomcat tomcat = new Tomcat();
    tomcat.setPort(8080);

    // 创建 Context
    Context context = tomcat.addContext("", "/tmp");

    // 注册 DispatcherServlet
    Tomcat.addServlet(context, "dispatcherServlet", new DispatcherServlet());
    context.addServletMappingDecoded("/", "dispatcherServlet");

    // 启动 Tomcat
    tomcat.start();    // 这不是外部 Tomcat，是 jar 包里的 Tomcat 类
    tomcat.getServer().await();
}
```

### 4.3 SpringApplication.run() 启动内嵌容器的流程

```
SpringApplication.run()
    │
    └── refreshContext(context)  // 刷新 Spring 上下文
            │
            └── onRefresh()     // AbstractApplicationContext
                    │
                    └── createWebServer()   // ServletWebServerApplicationContext
                            │
                            ├── 获取 ServletWebServerFactory
                            │   (TomcatServletWebServerFactory /
                            │    JettyServletWebServerFactory /
                            │    UndertowServletWebServerFactory)
                            │
                            └── webServer.start()  // 启动内嵌容器!
```

### 4.4 切换内嵌容器

```xml
<!-- 排除默认的 Tomcat -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<!-- 换成 Undertow -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

### 4.5 jar 包可以直接运行的原因

普通 jar 包不能直接运行（没有 main 方法的入口）。Spring Boot 使用 **Maven 插件** 打出的 jar 包结构特殊：

```
spring-boot-app.jar
  ├── META-INF/
  │     └── MANIFEST.MF          → Main-Class: JarLauncher
  ├── BOOT-INF/
  │     ├── classes/             → 你自己的代码
  │     └── lib/                 → 所有依赖 jar（包括 Tomcat!）
  └── org/springframework/boot/loader/
        └── JarLauncher.class    → Spring Boot 的类加载器，处理 BOOT-INF 结构
```

`JarLauncher` 会创建特殊的类加载器（`LaunchedURLClassLoader`），从 `BOOT-INF/lib/` 中加载嵌套的 jar 包，使 Tomcat 等内嵌容器的类可以正常加载。

### 4.6 面试标准回答

> "Spring Boot 不是把应用部署到外部 Tomcat，而是**把 Tomcat 作为普通 jar 依赖引入项目中**，在 `SpringApplication.run()` 启动流程中，通过 `ServletWebServerApplicationContext` 的 `onRefresh()` 方法编程式地创建并启动内嵌 Tomcat。这种方式的优势是：应用和容器打包在一起，`java -jar` 即可运行，不需要预先安装和配置 Tomcat 服务器。它的可执行 jar 包结构也特殊——通过 JarLauncher 和自定义类加载器来处理嵌套依赖 jar 的加载。"

---

## 五、配置文件 —— 应用的"遥控器"

### 5.1 两种配置格式

| 特性 | application.properties | application.yml |
|------|----------------------|-----------------|
| 语法 | `key=value`，扁平结构 | 缩进层级，树形结构 |
| 可读性 | 多级 key 冗长 | 层次清晰 |
| 功能 | 支持所有配置 | 支持所有配置 + 多文档 |
| 选择建议 | 简单项目够用 | 复杂配置推荐 |

**properties 写法**：
```properties
server.port=8080
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.username=root
spring.datasource.password=123456
```

**yml 写法**：
```yaml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/test
    username: root
    password: 123456
```

### 5.2 yml 的多文档特性

```yaml
# 第一个文档（默认配置）
server:
  port: 8080
spring:
  profiles:
    active: dev   # 激活 dev 环境

---
# 第二个文档（dev 环境）
spring:
  config:
    activate:
      on-profile: dev
server:
  port: 8080
---
# 第三个文档（prod 环境）
spring:
  config:
    activate:
      on-profile: prod
server:
  port: 80
```

### 5.3 多环境配置

```
resources/
  ├── application.yml              # 公共配置
  ├── application-dev.yml          # 开发环境
  ├── application-test.yml         # 测试环境
  └── application-prod.yml         # 生产环境
```

**激活方式（优先级从高到低）**：
1. 命令行参数：`java -jar app.jar --spring.profiles.active=prod`
2. 操作系统环境变量：`SPRING_PROFILES_ACTIVE=prod`
3. application.yml 中的 `spring.profiles.active`

### 5.4 @Value vs @ConfigurationProperties

| 特性 | @Value | @ConfigurationProperties |
|------|--------|-------------------------|
| **用途** | 注入单个属性 | 批量注入一组关联属性 |
| **松散绑定** | 不支持 | 支持（firstName = first-name = FIRST_NAME） |
| **JSR303 校验** | 不直接支持 | 支持 @Validated |
| **SpEL 表达式** | 支持 | 不支持 |
| **适用场景** | 零散的、个别的属性 | 一组有业务含义的配置 |

**@Value 示例**：
```java
@Value("${server.port}")
private int port;

@Value("${app.timeout:3000}")  // 默认值 3000
private long timeout;
```

**@ConfigurationProperties 示例**：
```java
@Component
@ConfigurationProperties(prefix = "app.datasource")
@Validated  // 支持校验
public class DataSourceConfig {
    @NotEmpty
    private String url;
    private String username;
    private String password;
    // getters / setters ...
}
```

```yaml
app:
  datasource:
    url: jdbc:mysql://localhost/db
    username: admin
    password: secret
```

> **选型建议**：第三方组件配置（如数据源、线程池）用 @ConfigurationProperties；业务代码中偶尔用一两个值的用 @Value。

### 5.5 配置优先级（从高到低）

```
1. 命令行参数（--server.port=9090）
2. SPRING_APPLICATION_JSON 环境变量
3. Servlet 相关参数
4. JNDI 属性
5. Java 系统属性（System.getProperties()）
6. 操作系统环境变量
7. application-{profile}.yml（外部 jar 包外）
8. application-{profile}.yml（内部 jar 包内）
9. application.yml（外部）
10. application.yml（内部）
11. @PropertySource 注解
12. SpringApplication.setDefaultProperties
```

**记忆口诀**：命（命令行）运（环境变量）外（外部配置）内（内部配置）个（个性化profile）默（默认）

---

## 六、Actuator 监控 —— 应用的"体检报告"

### 6.1 核心比喻

> **Actuator 是给应用装的"体检仪"**。它可以随时查看心跳（health）、各项指标（metrics）、查看日志级别（loggers）、查看运行环境信息（env），就像你体检时心电图、血常规、血压计一起上。

### 6.2 常用端点

| 端点 | 路径 | 作用 | 面试考点 |
|------|------|------|---------|
| `health` | `/actuator/health` | 应用健康状态 | 健康检查、K8s 探活 |
| `info` | `/actuator/info` | 应用基本信息 | 自定义 info 内容 |
| `metrics` | `/actuator/metrics` | 运行指标（JVM 内存、GC、HTTP 请求） | 对接 Prometheus + Grafana |
| `loggers` | `/actuator/loggers` | 查看和修改日志级别 | 线上动态调日志 |
| `env` | `/actuator/env` | 环境变量/配置属性 | 排查配置问题 |
| `beans` | `/actuator/beans` | 容器中所有 Bean | 排查 Bean 注入 |
| `mappings` | `/actuator/mappings` | 所有 URL 映射 | 排查接口冲突 |
| `threaddump` | `/actuator/threaddump` | 线程快照 | 排查死锁/性能 |
| `heapdump` | `/actuator/heapdump` | 堆内存快照 | OOM 排查 |
| `conditions` | `/actuator/conditions` | 自动配置条件报告 | 为啥某个自动配置没生效 |

### 6.3 配置示例

```yaml
# 暴露所有端点（生产环境应谨慎）
management:
  endpoints:
    web:
      exposure:
        include: "health,info,metrics,prometheus"
      base-path: /actuator    # 默认就是 /actuator
  endpoint:
    health:
      show-details: always     # 显示详细信息
  metrics:
    export:
      prometheus:
        enabled: true          # 启用 Prometheus 格式
```

### 6.4 对接 Prometheus + Grafana

```
Spring Boot App
    │
    ├── /actuator/metrics (Metrics 格式)
    │       ↓
    ├── /actuator/prometheus (Prometheus 格式，需要 micrometer-registry-prometheus)
    │       ↓
    └── Prometheus 定期拉取 → 存储到时序数据库
                                  ↓
                            Grafana 展示仪表盘
```

**引入依赖**：
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

### 6.5 自定义健康检查

```java
@Component
public class MyHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        // 检查某个依赖服务是否可用
        boolean isUp = checkExternalService();
        if (isUp) {
            return Health.up()
                .withDetail("service", "Available")
                .build();
        } else {
            return Health.down()
                .withDetail("service", "Unavailable")
                .build();
        }
    }
}
```

> 返回结果会聚合到 `/actuator/health` 中：
> ```json
> {
>   "status": "UP",
>   "components": {
>     "my": { "status": "UP", "details": {"service": "Available"} },
>     "diskSpace": { "status": "UP", ... },
>     "db": { "status": "UP", ... }
>   }
> }
> ```

### 6.6 面试标准回答

> "Actuator 是 Spring Boot 的生产级监控模块，提供了十几个开箱即用的端点。最核心的三个是：**health**（健康检查，可用于 K8s 的 liveness/readiness probe）、**metrics**（JVM 内存/GC/HTTP 请求等指标，可对接 Prometheus + Grafana 做可视化监控）、**loggers**（线上动态修改日志级别，方便问题排查）。在生产环境中，建议只暴露必要的端点，并通过 Spring Security 对 /actuator 路径进行权限控制。"

---

## 七、Spring Boot 启动流程 —— 跑车的"点火到起步"

### 7.1 核心比喻

> **Spring Boot 启动就像赛车发车流程**：
> 1. **Ecu 自检**（推断应用类型）—— 是 F1 赛车还是越野车？
> 2. **各项系统初始化**（加载 Initializer 和 Listener）—— 油路、电路、刹车检查
> 3. **引擎点火**（创建 ApplicationContext）—— 发动机启动
> 4. **热车/挂挡**（刷新上下文，加载 Bean）—— 挂挡、热车
> 5. **弹射起步**（启动内嵌容器）—— 松刹车，出发！

### 7.2 完整启动流程

```java
SpringApplication.run(Application.class, args)
```

```
Step 1: 创建 SpringApplication 实例
    │
    ├── 推断 Web 应用类型
    │    ├── REACTIVE (有 WebFlux 相关类)
    │    ├── SERVLET  (有 DispatcherServlet 相关类，没有 Reactive)
    │    └── NONE     (两者都没有)
    │
    ├── 加载 ApplicationContextInitializer (从 spring.factories)
    │    └── 在上下文刷新之前执行自定义初始化逻辑
    │
    └── 加载 ApplicationListener (从 spring.factories)
         └── 监听 Spring Boot 各阶段的事件

Step 2: 执行 run() 方法
    │
    ├── 2.1 启动 StopWatch 计时
    │
    ├── 2.2 创建 BootstrapContext
    │
    ├── 2.3 获取 SpringApplicationRunListeners
    │     └── 发布 ApplicationStartingEvent
    │
    ├── 2.4 准备 Environment
    │     ├── 解析命令行参数
    │     ├── 加载配置文件 (application.yml)
    │     └── 发布 ApplicationEnvironmentPreparedEvent
    │
    ├── 2.5 打印 Banner (那个大大的 SPRING 字母)
    │
    ├── 2.6 创建 ApplicationContext ★关键步骤★
    │     ├── 根据 Step1 推断的类型创建对应的 Context
    │     ├── Servlet   → AnnotationConfigServletWebServerApplicationContext
    │     ├── Reactive  → AnnotationConfigReactiveWebServerApplicationContext
    │     └── NONE      → AnnotationConfigApplicationContext
    │
    ├── 2.7 准备 ApplicationContext (prepareContext)
    │     ├── 设置 Environment
    │     ├── 执行 ApplicationContextInitializer
    │     ├── 注册启动类为 Bean
    │     └── 发布 ApplicationContextInitializedEvent
    │
    ├── 2.8 刷新 ApplicationContext ★最核心步骤★ (refreshContext)
    │     ├── 8.1 prepareRefresh()  — 准备工作（设置启动时间等）
    │     ├── 8.2 obtainFreshBeanFactory()  — 获取 BeanFactory
    │     ├── 8.3 prepareBeanFactory()  — 注册基础 Bean（环境、系统属性等）
    │     ├── 8.4 postProcessBeanFactory()  — BeanFactory 后处理
    │     ├── 8.5 invokeBeanFactoryPostProcessors()  — 执行 BeanFactoryPostProcessor
    │     │     └── 包括 **ConfigurationClassPostProcessor** 解析 @Configuration @Bean
    │     ├── 8.6 registerBeanPostProcessors()  — 注册 BeanPostProcessor
    │     │     └── 包括 **AutowiredAnnotationBeanPostProcessor** 处理 @Autowired
    │     ├── 8.7 initMessageSource()  — 国际化
    │     ├── 8.8 initApplicationEventMulticaster()  — 事件广播器
    │     ├── 8.9 onRefresh()  — ★启动内嵌 Web 服务器★
    │     │     └── createWebServer() → Tomcat.start()
    │     ├── 8.10 registerListeners()  — 注册监听器
    │     ├── 8.11 finishBeanFactoryInitialization()  — ★实例化所有单例 Bean★
    │     │     └── 非懒加载的单例 Bean 都在这步创建
    │     └── 8.12 finishRefresh()  — 发布 ContextRefreshedEvent
    │
    ├── 2.9 afterRefresh()  — 空实现，留给子类扩展
    │
    └── 2.10 发布 ApplicationStartedEvent / ApplicationReadyEvent
          └── 启动完成！
```

### 7.3 发布的事件顺序

| 事件 | 触发时机 | 用途 |
|------|---------|------|
| `ApplicationStartingEvent` | 启动刚开始 | 最早 - 可用于提前初始化 |
| `ApplicationEnvironmentPreparedEvent` | Environment 准备好 | 配置加载完成后 |
| `ApplicationContextInitializedEvent` | Context 创建后 | Context 刚创建 |
| `ApplicationPreparedEvent` | Context 刷新前 | Bean 还没加载 |
| `ApplicationStartedEvent` | Context 刷新后 | Bean 已加载，容器还没接受请求 |
| `ApplicationReadyEvent` | 所有就绪 | 应用已就绪，可以接受请求 |
| `ApplicationFailedEvent` | 启动失败 | 异常处理 |

### 7.4 面试标准回答

> "Spring Boot 的启动入口是 `SpringApplication.run()`，核心分为两大阶段：**创建 SpringApplication 实例**（推断应用类型、加载 Initializer 和 Listener）和**执行 run 方法**（准备环境、创建并刷新 ApplicationContext、启动内嵌容器）。
>
> 最核心的是 **refreshContext** 这一步——它复用了 Spring 的 `AbstractApplicationContext.refresh()` 12 步模板方法，其中关键步骤包括：`invokeBeanFactoryPostProcessors`（解析 @Configuration，触发自动配置）、`onRefresh`（在 `ServletWebServerApplicationContext` 中创建并启动内嵌 Tomcat）、`finishBeanFactoryInitialization`（实例化所有单例 Bean）。
>
> 需要特别注意的是 **自动配置入口** `AutoConfigurationImportSelector` 是在 `invokeBeanFactoryPostProcessors` 阶段被 `ConfigurationClassParser` 处理的——也就是说，自动配置类的加载和过滤发生在 Bean 实例化之前。"

---

## 八、热部署 —— devtools 的"不停机换轮胎"

### 8.1 核心比喻

> **热部署 = 改装赛车的"快拆轮胎"**：普通车换轮胎要上架、拆螺丝、换胎、装回去。改装赛车用快拆螺母，一秒换胎。Spring Boot DevTools 就是给你的应用装了"快拆螺母"，代码改了不用整机重启，只换改动的部分。

### 8.2 DevTools 原理：双类加载器机制

```
应用启动时，DevTools 创建两个类加载器：

    RestartClassLoader (restart 类加载器)
        │  加载：你自己写的代码（会变的那部分）
        │  └── com/yourcompany/**/*.class
        │
    BaseClassLoader (base 类加载器)
           加载：第三方 jar 包（不会变的那部分）
           └── spring-*.jar, hibernate-*.jar, etc.
```

**当检测到文件变化时**：
1. 只 **丢弃 RestartClassLoader**
2. 创建一个 **新的 RestartClassLoader**
3. 重新加载你自己的类
4. BaseClassLoader 中的 jar 包 **完全不重新加载**

> 这就解释了为什么 DevTools 重启比完全重启快得多——第三方库几千个类都不需要重新加载，只重新加载项目自己的几百个类。

### 8.3 配置和使用

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <optional>true</optional>  <!-- 生产环境不传递 -->
</dependency>
```

```yaml
spring:
  devtools:
    restart:
      enabled: true
      # 排除不需要触发热重启的目录
      exclude: static/**, templates/**
```

**IDEA 中的额外设置**：
1. File → Settings → Build → Compiler → 勾选 "Build project automatically"
2. Ctrl+Shift+Alt+/ → Registry → 勾选 "compiler.automake.allow.when.app.running"

### 8.4 面试标准回答

> "DevTools 热部署的核心是**双类加载器机制**。它把类分为两类：代码自己写的放在 RestartClassLoader 中（会变化），第三方 jar 放在 BaseClassLoader 中（不会变化）。当检测到文件变更时，只重建 RestartClassLoader，Base 完全不动。这样避免了全量加载第三方 jar，重启速度极快。
>
> 需要注意，DevTools 的'重启'和'热加载'是两个不同概念。代码逻辑变化需要重启（但比完全重启快），而静态资源/模板文件的变化是直接刷新生效的，不需要重启。"

---

## 九、拦截器 vs 过滤器（在 Spring Boot 中）

### 9.1 核心比喻

> **过滤器（Filter）= 小区门禁**：站在小区大门外，所有进出的人都要经过。它只认"通行证"（Request/Response），不管你来小区干嘛。
>
> **拦截器（Interceptor）= 单元楼门禁**：你已经进了小区（到了 DispatcherServlet），要去某个楼层找某个人（具体 Controller），门禁拦截你，检查你有没有"楼层权限"。

### 9.2 本质区别

| 维度 | Filter（过滤器） | Interceptor（拦截器） |
|------|-----------------|---------------------|
| **规范** | **Servlet 规范** | **Spring 框架** |
| **依赖容器** | 依赖 Servlet 容器（Tomcat） | 依赖 Spring IoC 容器 |
| **作用范围** | 所有请求（进入 Servlet 前） | 只拦截进入 DispatcherServlet 的请求 |
| **能否获取 Bean** | 可以，但需要特殊处理 | 天然可以，直接注入 |
| **粒度** | 只能访问 Request/Response | 可以访问 Handler（Controller 方法）、ModelAndView |
| **执行顺序** | Filter → Interceptor → AOP | 外 → 内 |
| **生命周期回调** | init() → doFilter() → destroy() | preHandle() → postHandle() → afterCompletion() |
| **静态资源** | 能拦截 | 不拦截（不进 DispatcherServlet） |

### 9.3 执行顺序

```
请求 → Filter1 → Filter2 → DispatcherServlet → Interceptor1 → Interceptor2 → Controller
                                                                                   │
                                                                                   ▼
响应 ← Filter1 ← Filter2 ← DispatcherServlet ← Interceptor1 ← Interceptor2 ← Controller
```

### 9.4 在 Spring Boot 中配置 Filter

**方式一：@WebFilter + @ServletComponentScan（简单场景）**：

```java
@WebFilter(urlPatterns = "/*")
public class MyFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        System.out.println("Filter: 小区门禁检查");
        chain.doFilter(request, response);
        System.out.println("Filter: 出门记录");
    }
}

// 启动类加 @ServletComponentScan
@SpringBootApplication
@ServletComponentScan
public class Application { ... }
```

**方式二：FilterRegistrationBean（推荐，可配置顺序）**：

```java
@Configuration
public class FilterConfig {
    @Bean
    public FilterRegistrationBean<MyFilter> myFilter() {
        FilterRegistrationBean<MyFilter> bean = new FilterRegistrationBean<>();
        bean.setFilter(new MyFilter());
        bean.addUrlPatterns("/api/*");
        bean.setOrder(1);  // 设置优先级，数字越小越先执行
        return bean;
    }
}
```

### 9.5 在 Spring Boot 中配置 Interceptor

```java
@Component
public class MyInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                             Object handler) {
        // 进入 Controller 之前
        System.out.println("Interceptor: 楼层门禁检查 - 即将访问: " + handler);
        return true;  // true=放行, false=拦截
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response,
                           Object handler, ModelAndView modelAndView) {
        // Controller 执行后，渲染视图前
        System.out.println("Interceptor: 楼层访客已离开");
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) {
        // 视图渲染完成后（无论是否异常都会执行）
        System.out.println("Interceptor: 访问记录归档");
    }
}

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired
    private MyInterceptor myInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(myInterceptor)
                .addPathPatterns("/api/**")    // 拦截的路径
                .excludePathPatterns("/api/login", "/api/register")  // 排除的路径
                .order(1);                     // 拦截顺序
    }
}
```

### 9.6 典型应用场景

| 需求 | 使用 Filter | 使用 Interceptor |
|------|-----------|-----------------|
| 编码过滤 | 首选 | - |
| 日志记录（全量请求） | 可以 | 更好（可注入 LogService） |
| 登录鉴权（Spring Security 以外） | 可以 | 首选（可获取 Controller 注解） |
| 权限校验 | - | 首选（可获取 @PreAuthorize 等） |
| 跨域 CORS | 首选（Spring 的 CorsFilter） | - |
| 性能统计 | 可以 | 更好（preHandle 记录时间，afterCompletion 计算耗时） |
| XSS 过滤 | 首选（修改 Request 内容） | 不太方便 |

### 9.7 面试标准回答

> "Filter 是 **Servlet 规范**定义的，在请求进入 Servlet 容器后、到达 DispatcherServlet 前执行，可以拦截所有请求包括静态资源。Interceptor 是 **Spring 框架**的，在经过 DispatcherServlet 分发后、到达 Controller 前执行，只能拦截进入 Spring MVC 的请求。
>
> 在 Spring Boot 中配置 Filter 推荐使用 `FilterRegistrationBean`（可以精确控制优先级），配置 Interceptor 则实现 `HandlerInterceptor` 接口并在 `WebMvcConfigurer.addInterceptors()` 中注册。
>
> 简单选择标准：需要修改 Request 内容（如 XSS 过滤、编码）用 Filter；需要感知 Controller 方法（如权限注解校验）用 Interceptor；需要注入 Spring Bean 方便就用 Interceptor。"

---

## 十、常见面试问题清单

| 序号 | 面试问题 | 答案要点 | 难度 |
|------|---------|---------|------|
| 1 | **Spring Boot 和 Spring 有什么区别？** | 自动配置、Starter、内嵌容器、Actuator；Spring Boot 是 Spring 的快速集成方案，不是新框架 | 基础 |
| 2 | **Spring Boot 自动配置的原理是什么？** | @SpringBootApplication → @EnableAutoConfiguration → AutoConfigurationImportSelector → spring.factories / .imports → 条件注解过滤 | 中等 |
| 3 | **@SpringBootApplication 注解包含哪三个注解？** | @SpringBootConfiguration、@EnableAutoConfiguration、@ComponentScan | 基础 |
| 4 | **Spring Boot 的 Starter 是什么？** | 依赖描述符、一站式依赖引入、版本仲裁、传递依赖管理 | 基础 |
| 5 | **Spring Boot 如何实现自动配置的条件过滤？** | @ConditionalOnClass（有类才配）、@ConditionalOnMissingBean（没自定义才配）、@ConditionalOnProperty（有配置才配）等 | 中等 |
| 6 | **Spring Boot 启动过程是怎样的？** | 推断应用类型 → 加载 Initializer/Listener → 准备 Environment → 创建并刷新 ApplicationContext（重点：自动配置加载、Bean 实例化、内嵌容器启动）→ 发布 Ready 事件 | 高频 |
| 7 | **Spring Boot 的内嵌 Tomcat 是如何工作的？** | Tomcat 作为 jar 依赖引入；SpringApplication.run() 中通过 ServletWebServerApplicationContext 编程式启动；JarLauncher 处理嵌套 jar 加载 | 中等 |
| 8 | **@Value 和 @ConfigurationProperties 的区别？** | 单个 vs 批量、不支持 vs 支持松散绑定、支持 SpEL vs 不支持、适用场景不同 | 中等 |
| 9 | **Filter 和 Interceptor 的区别？在 Spring Boot 中怎么配置？** | Servlet 规范 vs Spring；执行时机不同；FilterRegistrationBean vs WebMvcConfigurer.addInterceptors | 中等 |
| 10 | **如何实现 Spring Boot 项目的多环境配置？** | application-{profile}.yml、spring.profiles.active 指定、命令行/环境变量/配置文件三种激活方式 | 基础 |
| 11 | **Spring Boot DevTools 热部署的原理？** | 双类加载器机制：RestartClassLoader（项目代码）+ BaseClassLoader（第三方 jar），只重建前者 | 加分 |

---

## 十一、面试满分话术

### 话术一："请介绍一下 Spring Boot"

> "Spring Boot 是 Pivotal 团队（现 VMware）推出的基于 Spring Framework 的快速开发框架，核心设计哲学是**'约定大于配置'**。
>
> 它的核心能力可以概括为四个关键词：**自动配置**（根据类路径中的依赖自动装配 Spring Bean，比如引入了 MySQL 驱动和 DataSource 依赖就自动配好 DataSource）、**起步依赖**（Starter 机制实现一站式依赖管理，一个 spring-boot-starter-web 搞定 Web 开发所需的全部 jar 包，并且版本经过充分兼容性测试）、**内嵌容器**（将 Tomcat/Jetty/Undertow 作为普通依赖引入，打成一个可执行的 jar 包，java -jar 直接运行，不需要部署到外部应用服务器）、**Actuator 生产级监控**（提供 health、metrics、loggers 等端点，可无缝对接 Prometheus + Grafana 做云原生监控）。
>
> 需要强调的是，Spring Boot 不是替代 Spring，而是对 Spring 生态的**封装和增强**。它的底层仍然是 Spring 的 IoC 容器、AOP、Spring MVC，Spring Boot 的价值在于让你**用更少的配置、更快地集成这些技术**。对于微服务架构，Spring Cloud 全家桶也是建立在 Spring Boot 基础之上的。"

### 话术二："Spring Boot 的自动配置是怎么做的？有没有可能不生效？怎么排查？"

> "自动配置的原理可以这样理解：Spring Boot 启动时，`@SpringBootApplication` 注解中的 `@EnableAutoConfiguration` 通过 `@Import(AutoConfigurationImportSelector.class)` 导入选择器。这个选择器会去 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件（Spring Boot 2.x 是 spring.factories）中读取所有候选的自动配置类全限定名，然后用**条件注解**进行过滤——检查类路径是否有相关类、容器中是否已有用户自定义的 Bean、配置文件中是否有相关属性等——最终只让满足条件的自动配置类生效。
>
> **自动配置不生效的常见原因**有四个：
> 1. **没引入对应的 Starter 依赖**——类路径中没有相关类，@ConditionalOnClass 条件不满足；
> 2. **用户自己定义了同名 Bean**——@ConditionalOnMissingBean 发现容器中已存在，自动配置退缩；
> 3. **被 @SpringBootApplication 的 exclude 排除了**——通过 `@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})` 或 `spring.autoconfigure.exclude` 配置排除；
> 4. **配置文件中的属性不满足条件**——@ConditionalOnProperty 要求的属性值不对。
>
> **排查方法**：在 application.yml 中设置 `debug: true`，启动时会打印**自动配置报告**，明确告诉你哪些配置生效了、哪些没有生效以及原因。或者直接访问 `/actuator/conditions` 端点在线查看。"

### 话术三："说一下 Spring Boot 的启动流程"

> "Spring Boot 应用的启动入口是 `SpringApplication.run()`，整个流程可以分成**两个阶段、十个步骤**。
>
> **第一阶段：创建 SpringApplication 实例**
> - 推断应用类型（Servlet / Reactive / None）—— 通过检查类路径中是否有 DispatcherServlet 或 WebFlux 的类来判断
> - 从 spring.factories 中加载 ApplicationContextInitializer 和 ApplicationListener
> - 推断出 main 方法所在的类作为主配置类
>
> **第二阶段：执行 run() 方法**
> - 准备 Environment：加载 application.yml 和命令行参数，封装成 Environment 对象
> - 创建 ApplicationContext：根据第一阶段推断的类型创建对应上下文（Servlet 对应 AnnotationConfigServletWebServerApplicationContext）
> - 刷新 ApplicationContext（最核心的一步，复用 Spring 的 refresh() 12 步模板）：
>   - **invokeBeanFactoryPostProcessors**：解析 @Configuration 类，处理 @Import（包括 @EnableAutoConfiguration 导入的 AutoConfigurationImportSelector），加载并过滤所有自动配置类
>   - **onRefresh**：在 ServletWebServerApplicationContext 中创建并启动内嵌 Tomcat 容器
>   - **finishBeanFactoryInitialization**：实例化所有非懒加载的单例 Bean（@Component、@Service、@Bean 等）
> - 发布 ApplicationStartedEvent 和 ApplicationReadyEvent，标志应用启动完成
>
> 整个流程中，**自动配置的加载**（在 invokeBeanFactoryPostProcessors 中）和**内嵌容器的启动**（在 onRefresh 中）是 Spring Boot 区别于 Spring 的最关键步骤。"

---

## 十二、记忆口诀

### 口诀一：Spring Boot 核心四件套
> **自起内监**
> - **自**动配置（@EnableAutoConfiguration）
> - **起**步依赖（Starter）
> - **内**嵌容器（Tomcat in jar）
> - **监**控运维（Actuator）

### 口诀二：自动配置五步法
> **引包扫，条过生**
> - **引**入 @SpringBootApplication
> - **包**含 @EnableAutoConfiguration
> - **扫**描 spring.factories / .imports
> - **条**件注解过滤
> - **生**效生成 Bean

### 口诀三：配置优先级
> **命运外内个默**
> - 命（命令行参数）
> - 运（环境变量）
> - 外（外部配置文件）
> - 内（内部配置文件）
> - 个（个性化 profile）
> - 默（默认属性）

### 口诀四：Filter vs Interceptor 辩证
> **Filter 是门卫（Servlet），Interceptor 是管家（Spring）**
> - 门卫管全部进出（包括静态资源）
> - 管家只管主人起居（Controller）
> - 门卫看通行证（Request/Response）
> - 管家知道你要去哪间房（Handler）

### 口诀五：Starter 命名规范
> **官方左起春起步，三方右落春尾随**
> - 官方：`spring-boot-starter-xxx`（spring-boot 在左）
> - 三方：`xxx-spring-boot-starter`（spring-boot 在右）

---

## 十三、跨题串联

| 关联知识点 | 关联方式 | 对应笔记 |
|-----------|---------|---------|
| **SSM 框架** | Spring Boot = Spring + Spring MVC + MyBatis 的快速整合方案；传统 SSM 需要大量 XML 配置，Spring Boot 用自动配置 + 注解替代 | [[../../面试/SSM框架面试题]] |
| **微服务 / Spring Cloud** | Spring Boot 是 Spring Cloud 微服务体系的基石；Nacos、Gateway、Feign 等组件都基于 Spring Boot 自动配置机制 | [[../微服务/微服务完整篇]] |
| **JVM** | Actuator 的 metrics 端点暴露了 GC 次数、内存使用、线程数等 JVM 核心指标；内嵌容器的类加载机制涉及双亲委派 | [[../JVM/JVM完整篇]] |
| **Tomcat** | 传统外部 Tomcat 的部署架构（war + 多应用共享容器）vs Spring Boot 内嵌 Tomcat（一个应用一个容器，微服务友好） | [[Tomcat]] |
| **Docker / K8s** | Spring Boot 的 Fat Jar + 内嵌容器天然适合容器化；Actuator 的 health 端点就是为 K8s 探活准备的 | [[Docker与K8S]] |
| **MySQL / MyBatis** | 自动配置了 DataSource、SqlSessionFactory、事务管理器；数据源配置通过 spring.datasource.* 前缀统一管理 | [[../数据库/MySQL面试题]] |
| **Redis** | spring-boot-starter-data-redis 自动配置了 RedisTemplate 和 StringRedisTemplate；连接配置通过 spring.redis.* 前缀统一管理 | [[../Redis/Redis完整篇]] |
| **设计模式** | Spring Boot 大量使用了模板方法模式（refresh() 12 步）、工厂模式（ServletWebServerFactory）、观察者模式（ApplicationListener 事件机制） | [[../../设计模式/设计模式面试题]] |
| **Java 基础** | 理解 Spring Boot 需要：反射（注解解析）、动态代理（AOP）、SPI 机制（META-INF 下配置文件模式）、类加载器（双类加载器机制） | [[../Java基础/Java基础完整篇]] |

---

> **复习建议**：本笔记建议分两次复习。第一次主攻"自动配置原理"和"启动流程"（第 2、7 节），这是面试最高频考点。第二次复习"配置管理"、"Actuator"和"Filter vs Interceptor"（第 5、6、9 节），这是实操高频点。每次复习后对照"常见面试问题清单"自问自答，检验掌握程度。
