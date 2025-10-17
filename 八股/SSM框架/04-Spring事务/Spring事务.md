---
tags:
  - 八股
  - SSM
  - Spring
  - 事务
status: 待复习
created: 2026-05-02
---

# Spring 事务管理

> [!summary] 我的速记
> **@Transactional 本质是 AOP——在方法执行前开启事务，正常返回则提交，异常则回滚。**
> **传播行为 7 种，面试重点 3 种：REQUIRED（有则加入/无则新建）、REQUIRES_NEW（挂起旧事务/新建独立）、NESTED（嵌套，内层独立回滚）。**
> **事务失效 5 大场景：自调用、非public、异常被catch、rollbackFor 不匹配、传播类型不对。**

---

## 一、@Transactional 原理

```java
// @Transactional 本质 = TransactionInterceptor（AOP通知）
@Service
public class OrderService {
    @Transactional
    public void createOrder() {
        // 代理对象在执行前 → 开启事务
        orderMapper.insert(order);
        // 正常返回 → 提交事务
        // 抛异常 → 回滚事务（只回滚 RuntimeException 和 Error，除非配 rollbackFor）
    }
}
```

**底层链路：** `@Transactional` → `TransactionInterceptor.invoke()`（AOP 环绕通知） → `PlatformTransactionManager.getTransaction()` → 绑定 Connection 到 ThreadLocal → 执行方法 → 提交/回滚。

> 🧠 **事务的关键：** Spring 通过 ThreadLocal 保证同一个事务中使用同一个数据库连接（Connection）。

---

## 二、事务传播行为（7 种，重点 3 种）

| 传播行为             | 当前有事务      | 当前无事务 | 记忆口诀                |     |
| ---------------- | ---------- | ----- | ------------------- | --- |
| **REQUIRED**（默认） | 加入         | 新建    | **"必须要有"**——最常用     |     |
| **REQUIRES_NEW** | 挂起旧事务，新建独立 | 新建    | **"我就要新的"**——内外独立   |     |
| **NESTED**       | 嵌套子事务（保存点） | 新建    | **"嵌套"**——内层回滚不影响外层 |     |
| SUPPORTS         | 加入         | 不开启   | 有就用，没有拉倒            |     |
| NOT_SUPPORTED    | 挂起，非事务运行   | 不开启   | 强制非事务               |     |
| MANDATORY        | 加入         | 抛异常   | 必须有事务               |     |
| NEVER            | 抛异常        | 不开启   | 我就是不要事务             |     |

> 🧠 **REQUIRED vs REQUIRES_NEW vs NESTED 的区别：**
> ```
> REQUIRED：     外层回滚 → 内层也回滚   （一家人）
> REQUIRES_NEW： 外层回滚 → 内层独立已提交 （分家了，互相没关系）
> NESTED：       外层回滚 → 内层回滚      内层回滚 → 外层可以继续（父子关系，子错不影响父）
> ```

---

## 三、事务隔离级别 + 三大读问题

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 实现 |
|----------|------|-----------|------|------|
| READ_UNCOMMITTED | ✅ | ✅ | ✅ | 无锁 |
| READ_COMMITTED | ❌ | ✅ | ✅ | 快照读 / 行锁 |
| REPEATABLE_READ（MySQL默认） | ❌ | ❌ | ✅(部分) | MVCC + Gap Lock |
| SERIALIZABLE | ❌ | ❌ | ❌ | 表锁，串行执行 |

| 问题        | 描述                             | 例子                                  |
| --------- | ------------------------------ | ----------------------------------- |
| **脏读**    | 读到别的事务未提交的数据                   | 事务 A 改了没提交，B 读到了                    |
| **不可重复读** | 同一事务两次读到不同值（别的事务提交了UPDATE）     | 查询余额 100 → 别的事务改成 200 并提交 → 再查变 200 |
| **幻读**    | 同一事务两次查询结果集行数不同（别的事务提交了INSERT） | 查有 10 条记录 → 别的事务插入 1 条提交 → 再查 11 条  |
|           |                                |                                     |

---

## 四、事务失效 5 大场景

| #   | 场景                         | 原因                     | 解决                                                                              |
| --- | -------------------------- | ---------------------- | ------------------------------------------------------------------------------- |
| 1   | **自调用**                    | `this.methodB()` 不走代理  | 注入自己 / 拆到另一个 Bean                                                               |
| 2   | **非 public**               | Spring 默认只代理 public 方法 | 改成 public                                                                       |
| 3   | **异常被 catch 吃掉**           | 事务管理器感知不到异常            | 手动 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` 或抛出去 |
| 4   | **rollbackFor 不匹配**        | 默认只回滚 RuntimeException | `@Transactional(rollbackFor = Exception.class)`                                 |
| 5   | **传播类型错（如 NOT_SUPPORTED）** | 配置了非事务传播               | 确认传播行为                                                                          |

```java
// 场景3：异常被 catch → 事务不回滚！
@Transactional
public void save() {
    try {
        orderMapper.insert(order);
        int a = 1 / 0;  // 异常！
    } catch (Exception e) {
        log.error("出错了", e);
        // 吃了异常，事务管理器不知道 → 提交了！ ← 致命 bug
    }
}
```

---

## 常见面试问题清单

| #   | 问题                                   | 回答要点                                                             |
| --- | ------------------------------------ | ---------------------------------------------------------------- |
| Q1  | **@Transactional 原理？**               | AOP 环绕通知——TransactionInterceptor 在方法前后管理事务。通过 ThreadLocal 绑定同一连接 |
| Q2  | **REQUIRED/REQUIRES_NEW/NESTED 区别？** | REQUIRED 加入（一家人）；REQUIRES_NEW 新建独立（分家）；NESTED 嵌套（子错不影响父）         |
| Q3  | **事务隔离级别和三大读问题？**                    | 脏读（未提交）、不可重复读（UPDATE）、幻读（INSERT）。串行化全防，MySQL RR 通过 MVCC+间隙锁防     |
| Q4  | **事务什么时候失效？**                        | 自调用、非public、异常被catch、rollbackFor不匹配、传播类型错误                       |
| Q5  | **为什么不自调用？**                         | this 走原始对象，事务是代理对象管理的——切面不生效                                     |
|     |                                      |                                                                  |

---

## 面试满分话术

> **面试官：@Transactional 原理？传播行为和失效场景？**

> @Transactional 本质是用 AOP 实现的声明式事务。TransactionInterceptor 作为环绕通知，在方法前通过事务管理器开启事务（从连接池获取连接并绑定到 ThreadLocal），方法正常返回则提交，遇到 RuntimeException 或配了 rollbackFor 的异常则回滚。
>
> 传播行为面试重点三种：REQUIRED（默认，有则加入无则新建，最常用）、REQUIRES_NEW（挂起旧事务独立新开，内外互不影响）、NESTED（嵌套事务，内层回滚外层可继续）。
>
> 事务失效五个坑：自调用（this 不走代理）、非 public（Spring 不代理）、异常被 catch 吃掉了（事务管理器感知不到）、rollbackFor 默认只对 RuntimeException 生效、传播行为配错。

---

## 记忆口诀

```
事务靠 AOP，环绕通知前后管。
ThreadLocal 绑连接，同一事务同一链接。
REQUIRED 加入走，REQUIRES_NEW 独立开。
失效五种最常见：自调非公异常吃，回滚类配传播型。
```
