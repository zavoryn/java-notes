---
tags: [八股, java基础, BigDecimal]
status: 已完成
---

# BigDecimal 精确计算

> [!summary] 我的速记
> **为什么不用 float/double？** 二进制无法精确表示 0.1 等十进制小数（IEEE 754 精度丢失）。
> **三条铁律：** ① 用 `new BigDecimal("字符串")` 或 `BigDecimal.valueOf()`，**不能用** `new BigDecimal(double)`；② `divide` 必须指定 `RoundingMode`；③ 比较用 `compareTo()` 而非 `equals()`（`equals` 比较 scale，1.0≠1.00）。

## 1. 为什么不能用 float/double 表示精确金额？

```java
@Test
public void testFloatPrecision() {
    System.out.println(0.1 + 0.2);  // 输出: 0.30000000000000004
    System.out.println(1.0 - 0.9);  // 输出: 0.09999999999999998
}
```

> **根本原因**：计算机使用二进制存储，而很多十进制小数（如 0.1）是**二进制无限循环小数**，只能截断近似表示，导致精度丢失。

`float`（单精度，32位）和 `double`（双精度，64位）遵循 IEEE 754 浮点数标准，适合**科学计算**和**性能场景**，但**不适合表示货币等需要精确小数的场景**。

## 2. BigDecimal 的创建方式

```java
// ❌ 错误：从 double 构造，精度已经丢失
BigDecimal d = new BigDecimal(0.1);
// d = 0.1000000000000000055511151231257827021181583404541015625

// ✅ 正确：从 String 构造
BigDecimal s = new BigDecimal("0.1");
// s = 0.1（精确）

// ✅ 推荐：使用 valueOf（内部也是 String 转换）
BigDecimal v = BigDecimal.valueOf(0.1);
// v = 0.1（精确）
```

> [!important] 面试重点
> **永远用 `new BigDecimal(String)` 或 `BigDecimal.valueOf(double)` 创建 BigDecimal，不要用 `new BigDecimal(double)`！**
>
> `BigDecimal.valueOf(0.1)` 等价于 `new BigDecimal(Double.toString(0.1))`。

## 3. 常用方法

```java
BigDecimal a = new BigDecimal("10.5");
BigDecimal b = new BigDecimal("3.2");

// 加减乘除
BigDecimal sum = a.add(b);               // 13.7
BigDecimal diff = a.subtract(b);         // 7.3
BigDecimal product = a.multiply(b);      // 33.60

// ⚠️ divide 需要指定舍入模式，否则可能抛 ArithmeticException
BigDecimal quotient = a.divide(b, 2, RoundingMode.HALF_UP);  // 3.28

// 比较
a.compareTo(b);   // >0 表示 a > b; =0 表示相等; <0 表示 a < b
a.setScale(0, RoundingMode.HALF_UP);     // 设置小数位数
a.negate();                              // 取反：-10.5
a.abs();                                 // 绝对值
a.max(b);                                // 较大值
```

> ⚠️ `BigDecimal` 是**不可变类**，所有运算方法返回新对象，不会修改原对象。

## 4. 舍入模式（RoundingMode）

| 模式 | 描述 | 示例 (保留一位小数) |
|------|------|------|
| `HALF_UP` | 四舍五入（最常用） | 2.55 → 2.6 |
| `HALF_DOWN` | 五舍六入 | 2.55 → 2.5 |
| `UP` | 远离零取整（向上） | 2.51 → 2.6; -2.51 → -2.6 |
| `DOWN` | 向零取整（截断） | 2.59 → 2.5 |
| `CEILING` | 向正无穷取整 | 2.51 → 2.6; -2.51 → -2.5 |
| `FLOOR` | 向负无穷取整 | 2.59 → 2.5; -2.59 → -2.6 |
| `HALF_EVEN` | 银行家舍入（四舍六入五留双） | 2.55 → 2.6; 2.45 → 2.4 |
| `UNNECESSARY` | 不进行舍入，精度不对抛异常 | — |

> 金额计算中 **HALF_UP（四舍五入）** 最常用。

## 5. compareTo vs equals

```java
BigDecimal a = new BigDecimal("1.0");
BigDecimal b = new BigDecimal("1.00");

System.out.println(a.equals(b));      // false  ← scale 不同（1.0 是 1 位，1.00 是 2 位）
System.out.println(a.compareTo(b));   // 0      ← 数值相等
```

> [!important] 踩坑点
> `BigDecimal.equals()` 同时比较**数值和精度（scale）**，`1.0` 和 `1.00` 不相等。
> 比较数值大小时必须用 `compareTo()`！

## 6. 面试话术
- **二进制无法精确表示部分小数**：十进制小数（如 0.1）转换为二进制时会出现无限循环，计算机无法用二进制精确表示，导致精度丢失。
- **浮点数是近似值**：为解决二进制无法精确表示小数的问题，引入了 IEEE 标准，用近似值表示小数并引入精度概念，因此浮点数只是近似值，不能用于需要高精度的金额计算。
- **金额计算需高精度**：在支付类项目中，金额计算必须精确，否则会导致严重问题，因此不能使用浮点数表示金额。

> "涉及金融、金额等需要精确小数计算的场景，不能用 `float` 和 `double`，因为它们遵循 IEEE 754 浮点数标准，在二进制下无法精确表示某些十进制小数（如 0.1），会导致精度丢失。Java 提供了 `BigDecimal` 类来做精确计算，它使用 `BigInteger` 存储未缩放的值和一个 `int` 存储 scale（小数位数）。创建 BigDecimal 时必须用 String 构造器或 `valueOf()`，直接传 double 构造器会保留浮点数已有的精度误差。"

> "使用 BigDecimal 有几个常见坑：第一是 `new BigDecimal(double)` 会包含浮点误差，必须用 String 构造；第二是 `divide()` 不指定舍入模式时，遇到除不尽会抛 `ArithmeticException`；第三是 `equals()` 会比较 scale 导致 `1.0` 和 `1.00` 不相等，数值比较应该用 `compareTo()`。BigDecimal 是不可变类，所有运算都返回新对象。"

## 7. 记忆口诀

> **金额计算要精确，double float 别碰它。String 构造 valueOf，divide 必须加舍入。equals 比较看 scale，数值相等 compareTo。**

## 8. 示例代码汇总

```java
public class BigDecimalDemo {
    public static void main(String[] args) {
        // ✅ 正确创建
        BigDecimal price = new BigDecimal("19.99");
        BigDecimal quantity = BigDecimal.valueOf(3);

        // ✅ 金额计算
        BigDecimal total = price.multiply(quantity);                    // 59.97
        BigDecimal afterDiscount = total.multiply(new BigDecimal("0.8"));  // 47.976
        BigDecimal rounded = afterDiscount.setScale(2, RoundingMode.HALF_UP); // 47.98

        // ✅ 比较
        if (rounded.compareTo(BigDecimal.ZERO) > 0) {
            System.out.println("总价: " + rounded);
        }
    }
}
```
