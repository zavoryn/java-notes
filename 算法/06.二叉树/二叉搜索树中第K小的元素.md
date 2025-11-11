---
tags:
  - 算法/二叉树
  - leetcode/hot100
  - ACM模式
difficulty: 中等
leetcode: 230
---

# 二叉搜索树中第K小的元素

> [LeetCode 230. Kth Smallest Element in a BST](https://leetcode.cn/problems/kth-smallest-element-in-a-bst/)

## 题目

给定一个二叉搜索树的根节点 root，和一个整数 k，返回二叉搜索树中第 k 小的元素值。

```
输入：root = [3,1,4,null,2], k = 1
输出：1
```

## 解答

```java
static int count = 0, ans = 0;

public static int kthSmallest(TreeNode root, int k) {
    count = 0;
    inorder(root, k);
    return ans;
}

static void inorder(TreeNode root, int k) {
    if (root == null) return;
    inorder(root.left, k);
    count++;
    if (count == k) { ans = root.val; return; }
    inorder(root.right, k);
}
```

## 复杂度

- **时间**：O(n) 最坏，O(h+k) 平均
- **空间**：O(h)

## 关联题目

- [[验证二叉搜索树]] — BST 性质
- [[二叉树的中序遍历]] — 中序模板
