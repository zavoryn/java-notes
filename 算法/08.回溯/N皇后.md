---
tags:
  - 算法/回溯
  - leetcode/hot100
  - ACM模式
difficulty: 困难
leetcode: 51
---

# N 皇后

> [LeetCode 51. N-Queens](https://leetcode.cn/problems/n-queens/) ｜ [牛客 NC39](https://www.nowcoder.com/practice/c76402adc16a4f8b8c9b6e9b1e7c5e1a)

## 题目

按照国际象棋的规则，皇后可以攻击与之处在同一行、同一列或同一斜线上的棋子。

**n 皇后问题**研究的是如何将 n 个皇后放置在 n×n 的棋盘上，并且使皇后彼此之间不能相互攻击。

给你一个整数 n，返回所有不同的 n 皇后问题的解决方案。

```
输入：n = 4
输出：[[".Q..","...Q","Q...","..Q."],["..Q.","Q...","...Q",".Q.."]]
```

## ACM 输入格式

```
一行：n
```

```
4
```

## 思路

逐行放皇后，每行从第 0 列试到第 n-1 列，能放就放，放不了就回溯。

**核心**：放之前用 `isValid()` 扫一遍上方，确认不冲突。

```
棋盘的列、左上斜、右上斜都不能有皇后：

  \  |  /      ← 只检查当前行上方，因为下方还没放
   \ | /
  ← 当前位置
```

## 解答

```java
import java.util.*;

public class Main {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();

        List<List<String>> res = solveNQueens(n);
        System.out.print("[");
        for (int i = 0; i < res.size(); i++) {
            if (i > 0) System.out.print(",");
            System.out.print(res.get(i));
        }
        System.out.println("]");
        sc.close();
    }

    public static List<List<String>> solveNQueens(int n) {
        List<List<String>> res = new ArrayList<>();
        char[][] board = new char[n][n];
        for (char[] row : board) Arrays.fill(row, '.');
        backtrack(board, 0, res);
        return res;
    }

    static void backtrack(char[][] board, int row, List<List<String>> res) {
        int n = board.length;
        if (row == n) {
            List<String> sol = new ArrayList<>();
            for (char[] r : board) sol.add(new String(r));
            res.add(sol);
            return;
        }
        for (int col = 0; col < n; col++) {
            if (!isValid(board, row, col)) continue;

            board[row][col] = 'Q';
            backtrack(board, row + 1, res);
            board[row][col] = '.';
        }
    }

    // 检查在 (row, col) 放皇后是否冲突（只看上方，下方还没放过）
    static boolean isValid(char[][] board, int row, int col) {
        int n = board.length;
        // 同列上方
        for (int i = 0; i < row; i++) {
            if (board[i][col] == 'Q') return false;
        }
        // 左上斜
        for (int i = row - 1, j = col - 1; i >= 0 && j >= 0; i--, j--) {
            if (board[i][j] == 'Q') return false;
        }
        // 右上斜
        for (int i = row - 1, j = col + 1; i >= 0 && j < n; i--, j++) {
            if (board[i][j] == 'Q') return false;
        }
        return true;
    }
}
```

## 复杂度

- **时间**：O(n!) — 实际通过剪枝远小于此
- **空间**：O(n²)

---

## 踩坑记录

> ❓ `isValid` 为什么只检查上方，不检查下方？
>
> 棋盘是**逐行往下放**的，当前行下方的所有行都还是空的，还没放过皇后，所以不需要检查。

## 关联题目

- [[全排列]] — 同样是逐行/逐位决策 + used 标记 + 撤销
- [[子集]] — 回溯模板变体
