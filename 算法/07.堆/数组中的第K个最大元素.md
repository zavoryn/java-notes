---
tags:
  - 算法/堆
  - leetcode/hot100
  - ACM模式
difficulty: 中等
leetcode: 215
---

# 数组中的第K个最大元素

> [LeetCode 215. Kth Largest Element in an Array](https://leetcode.cn/problems/kth-largest-element-in-an-array/) ｜ [牛客 NC88](https://www.nowcoder.com/practice/e016ad9b7f0b45048c58a9f27ba618bf)

## 题目

给定整数数组 `nums` 和整数 `k`，请返回数组中**第 k 个最大的元素**。

```
输入：nums = [3,2,1,5,6,4], k = 2
输出：5
```

## ACM 输入格式

```
第一行：n k
第二行：n 个整数
```

```
6 2
3 2 1 5 6 4
```

## 解答

### 解法一：小顶堆（O(n log k)）

```java
import java.util.*;

public class Main {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt(), k = sc.nextInt();
        int[] nums = new int[n];
        for (int i = 0; i < n; i++) nums[i] = sc.nextInt();

        System.out.println(findKthLargest(nums, k));
        sc.close();
    }

    public static int findKthLargest(int[] nums, int k) {
        PriorityQueue<Integer> pq = new PriorityQueue<>(); // 小顶堆
        for (int x : nums) {
            pq.offer(x);
            if (pq.size() > k) pq.poll();
        }
        return pq.peek();
    }
}
```

### 解法二：快速选择（O(n) 期望）

```java
public static int findKthLargest(int[] nums, int k) {
    int target = nums.length - k; // 第K大 = 第 (n-k) 小（0-indexed）
    int left = 0, right = nums.length - 1;
    while (left < right) {
        int p = partition(nums, left, right);
        if (p == target) return nums[p];
        else if (p < target) left = p + 1;
        else right = p - 1;
    }
    return nums[left];
}

static int partition(int[] nums, int left, int right) {
    int pivot = nums[left];
    int i = left, j = right;
    while (i < j) {
        while (i < j && nums[j] >= pivot) j--;
        nums[i] = nums[j];
        while (i < j && nums[i] <= pivot) i++;
        nums[j] = nums[i];
    }
    nums[i] = pivot;
    return i;
}
```

## 复杂度

| 解法 | 时间 | 空间 |
|------|------|------|
| 小顶堆 | O(n log k) | O(k) |
| 快速选择 | O(n) 期望，O(n²) 最坏 | O(1) |

---

## 踩坑记录

> ❓ 为什么用小顶堆而不是大顶堆？
>
> 小顶堆维护**最大的 k 个**：堆顶是这 k 个里最小的，也就是第 k 大。大顶堆维护最小 k 个，不是这题目标。

## 关联题目

- [[前K个高频元素]] — 同样用小顶堆维护 TopK
- [[14.技巧/快速排序|快速排序]] — 快速选择 = 快排的 partition 只走一边
