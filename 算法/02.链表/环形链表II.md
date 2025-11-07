---
tags: [算法/链表, leetcode/hot100, ACM模式]
difficulty: 中等
leetcode: 142
---

# 环形链表 II

> [LeetCode 142. Linked List Cycle II](https://leetcode.cn/problems/linked-list-cycle-ii/)

## 题目

给定一个链表的头节点 head，返回链表开始入环的第一个节点。如果链表无环，返回 null。

## 解答

```java
public static ListNode detectCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next; fast = fast.next.next;
        if (slow == fast) {
            slow = head;
            while (slow != fast) { slow = slow.next; fast = fast.next; }
            return slow;
        }
    }
    return null;
}
```

## 踩坑

> Floyd 判圈两步：① 快慢相遇 → 有环；② 慢回 head，双指针同步走，再次相遇 = 入环点。
