---
tags: [算法/图论, leetcode/hot100, ACM模式]
difficulty: 中等
leetcode: 208
---

# 实现 Trie (前缀树)

> [LeetCode 208. Implement Trie (Prefix Tree)](https://leetcode.cn/problems/implement-trie-prefix-tree/)

## 题目

实现 Trie 类：`insert(word)`、`search(word)`、`startsWith(prefix)`。

## 解答

```java
static class Trie {
    Trie[] children = new Trie[26];
    boolean isEnd = false;

    public void insert(String word) {
        Trie node = this;
        for (char c : word.toCharArray()) {
            int idx = c - 'a';
            if (node.children[idx] == null) node.children[idx] = new Trie();
            node = node.children[idx];
        }
        node.isEnd = true;
    }

    public boolean search(String word) {
        Trie node = find(word);
        return node != null && node.isEnd;
    }

    public boolean startsWith(String prefix) {
        return find(prefix) != null;
    }

    Trie find(String s) {
        Trie node = this;
        for (char c : s.toCharArray()) {
            node = node.children[c - 'a'];
            if (node == null) return null;
        }
        return node;
    }
}
```
