---
tags:
  - 算法
  - ACM模式
created: 2026-04-29
---

# ACM 模式要点

> 刷题过程中反复踩坑的写法，统一记在这里。

## 输出格式

ACM 输出到标准输出，**不要**返回数组对象、不要带 `[]` 和逗号。

```java
// 输出 List<List<Integer>> 结果
for (List<Integer> triple : result) {
    System.out.println(triple.get(0) + " " + triple.get(1) + " " + triple.get(2));
}

// 输出 int[] 结果
System.out.println(result[0] + " " + result[1]);
```

## 快捷构造 List

```java
// 推荐：一行搞定
ans.add(Arrays.asList(a, b, c));

// 不推荐：
List<Integer> tmp = new ArrayList<>();
tmp.add(a);
tmp.add(b);
tmp.add(c);
ans.add(tmp);
```

## 两数之和 HashMap 模板

```java
// 易错：先检查再 put；找到后 break；别漏 put
for (int i = 0; i < n; i++) {
    if (map.containsKey(target - nums[i])) {
        System.out.println(map.get(target - nums[i]) + " " + i);
        break;
    }
    map.put(nums[i], i);
}
```

## 关联题目

- [[01.数组与字符串/两数之和]] — ACM 输入输出入门
- [[01.数组与字符串/三数之和]] — List 输出、Arrays.asList 用法
