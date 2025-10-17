---
tags:
  - 八股
  - SSM
  - MyBatis
status: 待复习
created: 2026-05-02
---

# MyBatis 核心原理

> [!summary] 我的速记
> **执行流程：SqlSession → Executor → StatementHandler → JDBC。**
> **#{} vs ${}：** `#{}` 是预编译占位符（防 SQL 注入），`${}` 是字符串拼接（危险，仅用于动态表名/排序字段）。
> **两级缓存：** 一级缓存（SqlSession 级别，默认开启）→ 二级缓存（Mapper 级别，跨 Session，需手动开启）。

---

## 一、MyBatis 执行流程（面试必画）

```
① 读取配置文件 → SqlSessionFactoryBuilder → SqlSessionFactory
  │
  ├── ② SqlSessionFactory.openSession() → 获取 SqlSession
  │
  ├── ③ SqlSession.getMapper(UserMapper.class) → JDK 动态代理生成 Mapper 代理对象
  │
  ├── ④ 调用 mapper.findById(1)
  │     └→ MapperProxy.invoke()
  │        └→ SqlSession.selectOne()
  │           └→ Executor.query()              ← 核心执行器
  │              ├── 一级缓存查询
  │              ├── 二级缓存查询（如果开启）
  │              └→ 缓存未命中
  │                 └→ StatementHandler         ← 创建 JDBC Statement
  │                    ├── ParameterHandler     ← 参数映射（#{} → ?）
  │                    ├── JDBC 执行 SQL
  │                    └── ResultSetHandler     ← 结果映射（ResultSet → Java对象）
  │
  └── ⑤ 返回结果 → 关闭 SqlSession
```

> 🧠 **核心：** MyBatis 通过 JDK 动态代理为每个 Mapper 接口生成代理对象。你调 mapper.findById() → 代理对象拦截 → 拿方法名找到对应的 SQL → 通过 Executor 走 JDBC 执行。

---

## 二、#{} vs ${}——面试必考

| | `#{}` | `${}` |
|---|---|---|
| **处理方式** | **预编译占位符 ?** | **字符串直接拼接** |
| **SQL 注入** | ✅ 安全 | ❌ 危险——用户可以注入恶意 SQL |
| **自动加引号** | ✅ 字符串类型自动加 `' '` | ❌ 不加，需要手动加 |
| **使用场景** | **WHERE 条件中的值**（绝大多数场景） | 动态表名、动态字段名、ORDER BY |

```xml
<!-- ✅ #{}：预编译，安全 -->
<select id="findByName" resultType="User">
    SELECT * FROM user WHERE name = #{name}
    <!-- 生成：SELECT * FROM user WHERE name = ?  → 安全！-->
</select>

<!-- ❌ ${}：字符串拼接，SQL 注入风险 -->
<select id="findByName" resultType="User">
    SELECT * FROM user WHERE name = '${name}'
    <!-- 用户输入: ' OR '1'='1 → SELECT * FROM user WHERE name = '' OR '1'='1' → 全表泄露！-->
</select>

<!-- ✅ ${} 正确用法：动态表名/字段名/排序（#{} 无法用于这些场景） -->
<select id="findAll" resultType="User">
    SELECT * FROM ${tableName} ORDER BY ${orderColumn} ${orderDirection}
</select>
```

> 🧠 **为什么 #{} 能防 SQL 注入？** 因为它是预编译——先发送 SQL 模板（`WHERE name = ?`），再把参数值传过去。参数值**永远不会被当成 SQL 代码执行**，只是数据。

---

## 三、一级缓存 vs 二级缓存

| | 一级缓存 | 二级缓存 |
|---|---|---|
| **级别** | **SqlSession** | **Mapper（namespace）** |
| **默认开启** | ✅ 默认开启 | ❌ 需手动配置 |
| **作用范围** | 同一个 SqlSession 内 | 跨 SqlSession，同一 Mapper |
| **失效条件** | 执行 insert/update/delete、commit、close | 同上，所有关联 Mapper 的缓存都会清 |
| **问题** | 脏读（其他 SqlSession 改了数据，缓存不更新） | 更大范围的脏读 |

```java
// 一级缓存的脏读问题
SqlSession session1 = factory.openSession();
User u1 = session1.selectOne("findById", 1);   // 查库 → 缓存

SqlSession session2 = factory.openSession();
session2.update("updateName", ...);             // session2 改数据 → 提交
session2.close();

User u2 = session1.selectOne("findById", 1);   // 走了缓存！拿到旧数据！
```

> 🧠 **生产环境一般关闭二级缓存。** 原因：①缓存数据可能过时（其他服务改了库）；②分布式环境下缓存不一致；③不如直接用 Redis 做缓存层。

---

## 四、MyBatis 插件机制（四大拦截器）

基于**责任链模式 + JDK 动态代理**：

| 拦截器 | 拦截对象 | 可以做什么 |
|--------|---------|-----------|
| **Executor** | 执行器 | 分页（PageHelper）、读写分离、二级缓存 |
| **ParameterHandler** | 参数处理器 | 参数加密/解密 |
| **StatementHandler** | SQL 预处理器 | 分页（改 SQL）、SQL 拦截 |
| **ResultSetHandler** | 结果集处理器 | 结果脱敏、结果二次处理 |

```java
@Intercepts(@Signature(type = StatementHandler.class, method = "prepare", args = {Connection.class, Integer.class}))
public class SqlInterceptor implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        StatementHandler handler = (StatementHandler) invocation.getTarget();
        String sql = handler.getBoundSql().getSql();
        System.out.println("SQL: " + sql);  // 打印 SQL
        return invocation.proceed();        // 放行
    }
}
```

---

## 常见面试问题清单

| # | 问题 | 回答要点 |
|---|------|----------|
| Q1 | **MyBatis 执行流程？** | SqlSession→Executor→StatementHandler→JDBC→ResultSetHandler。Mapper 通过 JDK 动态代理生成 |
| Q2 | **#{} vs ${}？** | #{}预编译占位符(安全)，${}字符串拼接(危险)。${}仅用于动态表名/字段名 |
| Q3 | **一级缓存和二级缓存？** | 一级默认开启(SqlSession内)，二级需手动(Mapper级)。生产一般关二级用 Redis |
| Q4 | **MyBatis 怎么连 Spring？** | SqlSessionFactoryBean 生成 SqlSessionFactory → @MapperScan 扫 Mapper 接口 → 生成代理对象注入 |
| Q5 | **分页插件原理？** | PageHelper 拦截 Executor.query() → 改 SQL，加上 LIMIT → 执行前再加 count 查询 |
| Q6 | **插件机制是什么？** | 责任链+动态代理。四大拦截器可拦截：Executor、ParameterHandler、StatementHandler、ResultSetHandler |

---

## 面试满分话术
- **核心定义**：MyBatis 是将 SQL 和 Java 对象通过映射框架定义、并通过动态代理执行的工具。
- **初始化阶段**：项目启动时，
- MyBatis 解析 MyBatis-config.xml，加载所有 mapper 文件，
- 将数据源、插件、类型别名等注入核心对象configuration，每条 SQL 语句转为 mapper的 statement 存入，
- 最终打包生成核心工厂类 SqlSessionFactory。
- **执行阶段**：先获取 SqlSession，
- 调用无实现类的 mapper方法时，、
- MyBatis 生成代理对象 methodProxy；
- methodProxy 从 configuration 中找到对应的 methodStatement，
- 交给 executor 执行。
- **执行细节**：先查缓存
- ```
 Executor.query()  ← 先查缓存
           ├── 一级缓存（SqlSession 内）命中 → 直接返回
           ├── 二级缓存（Mapper 级别）命中 → 直接返回
           └── 缓存未命中
              └→ StatementHandler → 创建 JDBC Statement
                 └→ ParameterHandler → #{} → ? → 设参数
                    └→ JDBC 执行 SQL
                       └→ ResultSetHandler → ResultSet → Jav
```
- 执行时由 statementHandle 准备参数，parameterHandle 绑定参数，setHandle 映射结果；事务提交、回滚、关闭链接由 SqlSession 统一控制。

> **面试官：MyBatis 执行流程？#{} 和 ${} 的区别？**

> MyBatis 执行流程从 SqlSession 出发，通过 Executor 执行 SQL。Executor 先查缓存（一级/二级），缓存未命中则委托给 StatementHandler——它负责创建 JDBC Statement、调用 ParameterHandler 做参数绑定（#{}→?）、执行 SQL、再通过 ResultSetHandler 把 ResultSet 映射成 Java 对象。
>
> #{} 是预编译占位符——SQL 模板先发给数据库编译（SELECT * FROM user WHERE id = ?），再把参数值传过去。参数值永远是数据而非代码，所以能防 SQL 注入。${} 是字符串直接拼接——参数会拼进 SQL 语句中，有 SQL 注入风险。${} 唯一的合理使用场景是动态表名和 ORDER BY 字段（这些地方预编译占位符不能用）。
>
> MyBatis 有两级缓存：一级默认开启（SqlSession 内有效，脏读风险），二级需手动配置（跨 Session，Mapper 级别）。生产环境一般关闭二级缓存，统一用 Redis 做缓存层。分页插件如 PageHelper 就是拦截 Executor.query()，在执行前修改 SQL 加上 LIMIT。

---

## 记忆口诀

```
SqlSession 是入口，Executor 来执行。
StatementHandler 建 JDBC，ResultSet 映射回对象。
#号预编译安全，$号拼接防注入。
一级缓存 Session 内，二级 Mapper 需手开。
生产关掉用 Redis，分页插件拦 Executor。
```
