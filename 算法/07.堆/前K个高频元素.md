---
tags:
  - 算法/堆
  - leetcode/hot100
  - ACM模式
difficulty: 中等
leetcode: 347
---

# 前K个高频元素

> [LeetCode 347. Top K Frequent Elements](https://leetcode.cn/problems/top-k-frequent-elements/) ｜ [牛客 NC97](https://www.nowcoder.com/practice/e8a1e7d50b3942e4a5e1e6a1e1e1e1e1)

## 题目

给你一个整数数组 `nums` 和一个整数 `k`，请你返回其中出现频率**前 k 高**的元素。你可以按任意顺序返回答案。

```
输入：nums = [1,1,1,2,2,3], k = 2
输出：[1,2]
```

## ACM 输入格式

```
第一行：n k
第二行：n 个整数
```

```
6 2
1 1 1 2 2 3
```

## 解答

```java
import java.util.*;

public class Main {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt(), k = sc.nextInt();
        int[] nums = new int[n];
        for (int i = 0; i < n; i++) nums[i] = sc.nextInt();

        int[] res = topKFrequent(nums, k);
        for (int i = 0; i < res.length; i++) {
            if (i > 0) System.out.print(" ");
            System.out.print(res[i]);
        }
        sc.close();
    }

    public static int[] topKFrequent(int[] nums, int k) {
        // 1. 统计频率
        Map<Integer, Integer> freq = new HashMap<>();
        for (int x : nums) freq.put(x, freq.getOrDefault(x, 0) + 1);

        // 2. 小顶堆，按频率排序
        PriorityQueue<Integer> pq = new PriorityQueue<>((a, b) -> freq.get(a) - freq.get(b));
        for (int key : freq.keySet()) {
            pq.offer(key);
            if (pq.size() > k) pq.poll();
        }

        // 3. 收集结果
        int[] res = new int[k];
        for (int i = k - 1; i >= 0; i--) res[i] = pq.poll();
        return res;
    }
}
```

## 复杂度

- **时间**：O(n log k)
- **空间**：O(n)

---

## 踩坑记录

> ❓ 堆排序用 `(a, b) -> freq.get(a) - freq.get(b)` 会溢出吗？
>
> 频率差不会超过 n，一般不会溢出。但更安全的写法是 `Comparator.comparingInt(freq::get)`。

## 关联题目

- [[数组中的第K个最大元素]] — 同样是「小顶堆维护 TopK」
- [[字母异位词分组]] — HashMap 统计特征
