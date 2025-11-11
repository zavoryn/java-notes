---
tags:
  - 算法/二叉树
  - leetcode/hot100
  - ACM模式
difficulty: 中等
leetcode: 437
---

# 路径总和 III

> [LeetCode 437. Path Sum III](https://leetcode.cn/problems/path-sum-iii/)

## 题目

给定一个二叉树的根节点 root，和一个整数 targetSum，求该二叉树里节点值之和等于 targetSum 的路径数目。路径不需要从根节点开始，也不需要在叶子节点结束，但必须从上到下。

```
输入：root = [10,5,-3,3,2,null,11,3,-2,null,1], targetSum = 8
输出：3
```

## 解答

```java
static int count = 0;

public static int pathSum(TreeNode root, int targetSum) {
    Map<Long, Integer> prefix = new HashMap<>();
    prefix.put(0L, 1);
    dfs(root, 0L, targetSum, prefix);
    return count;
}

static void dfs(TreeNode root, long curSum, int target, Map<Long, Integer> prefix) {
    if (root == null) return;
    curSum += root.val;
    count += prefix.getOrDefault(curSum - target, 0);
    prefix.put(curSum, prefix.getOrDefault(curSum, 0) + 1);
    dfs(root.left, curSum, target, prefix);
    dfs(root.right, curSum, target, prefix);
    prefix.put(curSum, prefix.get(curSum) - 1); // 回溯
}
```

## 复杂度

- **时间**：O(n)
- **空间**：O(n)

## 踩坑记录

> 核心是前缀和 + 回溯：`curSum - target` 在 map 中出现几次，就有几条以当前节点结尾的有效路径。离开当前节点时要 `prefix.put(curSum, count-1)` 撤销。
