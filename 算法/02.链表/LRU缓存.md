---
tags: [算法/链表, leetcode/hot100, ACM模式]
difficulty: 中等
leetcode: 146
---

# LRU 缓存

> [LeetCode 146. LRU Cache](https://leetcode.cn/problems/lru-cache/)

## 题目

实现 LRUCache 类：`get(key)` 返回 value 或 -1；`put(key, value)` 插入或更新。容量满时淘汰最久未使用的。

## 解答

```java
static class LRUCache {
    class Node { int key, val; Node prev, next; Node(int k, int v) { key = k; val = v; } }
    Map<Integer, Node> map = new HashMap<>();
    Node head = new Node(0, 0), tail = new Node(0, 0);
    int cap;

    public LRUCache(int capacity) {
        cap = capacity; head.next = tail; tail.prev = head;
    }

    public int get(int key) {
        Node node = map.get(key);
        if (node == null) return -1;
        moveToHead(node);
        return node.val;
    }

    public void put(int key, int value) {
        Node node = map.get(key);
        if (node != null) { node.val = value; moveToHead(node); return; }
        if (map.size() == cap) { Node last = tail.prev; remove(last); map.remove(last.key); }
        Node n = new Node(key, value); map.put(key, n); addToHead(n);
    }

    void remove(Node n) { n.prev.next = n.next; n.next.prev = n.prev; }
    void addToHead(Node n) { n.next = head.next; n.prev = head; head.next.prev = n; head.next = n; }
    void moveToHead(Node n) { remove(n); addToHead(n); }
}
```

## 复杂度: O(1) per operation
