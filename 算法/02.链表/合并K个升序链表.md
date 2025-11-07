---
tags:
  - 算法/链表
  - leetcode/hot100
  - ACM模式
difficulty: 困难
leetcode: 23
---

# 合并K个升序链表

> [LeetCode 23. Merge k Sorted Lists](https://leetcode.cn/problems/merge-k-sorted-lists/) ｜ [牛客 NC51 ACM 练习](https://www.nowcoder.com/practice/65cfde9e5b9b4cf2b6bafa5f3ef33fa6)

## 题目

给你一个链表数组，每个链表都已经按升序排列。将所有链表合并到一个升序链表中，返回合并后的链表。

```
输入：lists = [[1,4,5],[1,3,4],[2,6]]
输出：[1,1,2,3,4,4,5,6]
```

## ACM 输入格式

```
第一行：k（链表个数）
接下来 k 行：每行第一个数为链表长度 m，后面 m 个为节点值
```

```
3
3 1 4 5
3 1 3 4
2 2 6
```

## 解答

```java
import java.util.*;

public class Main {
    static class ListNode {
        int val;
        ListNode next;
        ListNode(int x) { val = x; }
    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);

        int k = sc.nextInt();
        ListNode[] lists = new ListNode[k];
        for (int i = 0; i < k; i++) {
            int m = sc.nextInt();
            ListNode dummy = new ListNode(0);
            ListNode cur = dummy;
            for (int j = 0; j < m; j++) {
                cur.next = new ListNode(sc.nextInt());
                cur = cur.next;
            }
            lists[i] = dummy.next;
        }

        ListNode result = mergeKLists(lists);

        while (result != null) {
            System.out.print(result.val);
            if (result.next != null) System.out.print(" ");
            result = result.next;
        }
        sc.close();
    }

    public static ListNode mergeKLists(ListNode[] lists) {
        PriorityQueue<ListNode> pq = new PriorityQueue<>((a, b) -> a.val - b.val);
        for (ListNode head : lists) {
            if (head != null) pq.offer(head);
        }
        ListNode dummy = new ListNode(0);
        ListNode cur = dummy;
        while (!pq.isEmpty()) {
            ListNode node = pq.poll();
            cur.next = node;
            cur = cur.next;
            if (node.next != null) {
                pq.offer(node.next);
            }
        }
        return dummy.next;
    }
}
```

## 复杂度

- **时间**：O(N log k) — N 为总节点数，堆中最多 k 个元素，每次 poll/offer O(log k)
- **空间**：O(k) — 优先队列最多存 k 个节点

---

## 关联题目

- [[反转链表]] — 链表基础操作
- [[环形链表]] — 链表遍历 + 双指针
- 合并两个有序链表 — 本题的基础操作 `mergeTwo`
