---
tags:
  - 算法/贪心
  - leetcode/hot100
  - ACM模式
difficulty: 中等
leetcode: 45
---

# 跳跃游戏 II

> [LeetCode 45. Jump Game II](https://leetcode.cn/problems/jump-game-ii/)

## 题目

每个元素表示从该位置跳跃的最大长度，返回到达最后一个位置的最小跳跃次数。保证可达。

```
输入：nums = [2,3,1,1,4]
输出：2
```

## 解答

```java
public int jump(int[] nums) {
        if(nums.length==1) return 0;

        int curEnd=0;
        int nextEnd=0;
        int step=0;

        for(int i=0;i<nums.length;i++){
            nextEnd=Math.max(nextEnd,i+nums[i]);

            if(i==curEnd){
                step++;
                curEnd=nextEnd;

                if(curEnd>=nums.length-1) break;
            }
        }
        return step;
    }
```

## 复杂度

- **时间**：O(n)
- **空间**：O(1)

## 复习标记

> 🔁 2026-05-13 思路遗忘，下次重点复习：贪心边界法 `curEnd` / `curFarthest`
