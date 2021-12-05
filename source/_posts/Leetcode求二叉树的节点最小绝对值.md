---
title: leetcode求二叉树的节点最小绝对值
date: 2018-07-25 23:36:59
tags: [leetcode,算法]
categories: [算法,leetcode]
---

# 二叉树的中序遍历
题目如下:
```java
Given a binary search tree with non-negative values, find the minimum absolute difference between values of any two nodes.

Example:

Input:

   1
    \
     3
    /
   2

Output:
1

Explanation:
The minimum absolute difference is 1, which is the difference between 2 and 1 (or between 2 and 3).
```

意思就是需要求出一个二叉树中绝对值最小的值。

刚开始的第一个想法就是递归，然后比较当前节点和其左右两个节点的值，但是发现有一个缺陷就是如果一个根节点是3，其左节点是10，10的右节点是4，那么最小的值便是1，然后通过递归只能是 10-3，10-4 。所以当时就放弃了

## 中序递归

后来又想到了中序递归便是这种遍历顺序，也就是通过中序遍历的方式便可以正确的得出结果了。但是一直想不出这种情况如何进行中序操作，但是在讨论区发现了一个解法如下：
```java

   int minDiff = Integer.MAX_VALUE;
   TreeNode prev;
   public int getMinimum(TreeNode root) {
        inorder(root);
        return minDiff;
    }

    public void inorder(TreeNode root) {
        if (root == null) return;
        inorder(root.left);
        if (prev != null) minDiff = Math.min(minDiff, root.val - prev.val);
        prev = root;
        inorder(root.right);
    }
```

后来发现这个 prev 才是这个中序遍历的关键，当一个节点的左节点遍历完之后，保存该节点为 Prev 然后与下一个节点进行比较。

![](流程图.png)