---
tags: [算法/链表, leetcode/hot100, ACM模式]
difficulty: 中等
leetcode: 19
---

# 删除链表的倒数第N个结点

> [LeetCode 19. Remove Nth Node From End of List](https://leetcode.cn/problems/remove-nth-node-from-end-of-list/)

## 解答

```java
public static ListNode removeNthFromEnd(ListNode head, int n) {
    ListNode dummy = new ListNode(0); dummy.next = head;
    ListNode fast = dummy, slow = dummy;
    for (int i = 0; i <= n; i++) fast = fast.next;
    while (fast != null) { slow = slow.next; fast = fast.next; }
    slow.next = slow.next.next;
    return dummy.next;
}
```

> 快指针先走 n+1 步，然后同步走，慢指针落在待删节点的前一个。
