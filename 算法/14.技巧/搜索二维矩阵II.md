---
tags: [算法/矩阵, leetcode/hot100, ACM模式]
difficulty: 中等
leetcode: 240
---

# 搜索二维矩阵 II

> [LeetCode 240. Search a 2D Matrix II](https://leetcode.cn/problems/search-a-2d-matrix-ii/)

## 题目

编写一个高效的算法来搜索 m x n 矩阵 matrix 中的一个目标值 target。该矩阵每行从左到右升序，每列从上到下升序。

```
输入：matrix = [[1,4,7,11,15],[2,5,8,12,19],[3,6,9,16,22]], target = 5
输出：true
```

## 解答

```java
public static boolean searchMatrix(int[][] matrix, int target) {
    int i = 0, j = matrix[0].length - 1;  // 从右上角开始
    while (i < matrix.length && j >= 0) {
        if (matrix[i][j] == target) return true;
        if (matrix[i][j] < target) i++;    // 排除这一行
        else j--;                           // 排除这一列
    }
    return false;
}
```

> Z 字查找：从右上角，小于 target 下移，大于 target 左移。O(m+n)。
