---
tags: [算法/前缀和, leetcode/hot100, ACM模式]
difficulty: 中等
leetcode: 560
---

# 和为K的子数组

> [LeetCode 560. Subarray Sum Equals K](https://leetcode.cn/problems/subarray-sum-equals-k/)

## 题目

给你一个整数数组 nums 和一个整数 k，请你统计并返回该数组中和为 k 的连续子数组的个数。

```
输入：nums = [1,1,1], k = 2
输出：2
```

## 解答

```java
public static int subarraySum(int[] nums, int k) {
    Map<Integer, Integer> prefix = new HashMap<>();
    prefix.put(0, 1);
    int sum = 0, count = 0;
    for (int x : nums) {
        sum += x;
        count += prefix.getOrDefault(sum - k, 0);
        prefix.put(sum, prefix.getOrDefault(sum, 0) + 1);
    }
    return count;
}
```

> 前缀和 + HashMap：`sum[i] - sum[j] = k` → `sum[j] = sum[i] - k`。
