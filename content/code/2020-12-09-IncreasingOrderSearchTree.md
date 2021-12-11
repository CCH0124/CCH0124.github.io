---
title: 897. Increasing Order Search Tree
date: 2020-12-09
description: "in-Order Traversal"
tags: [LeetCode]
draft: false
---

[題目](https://leetcode.com/problems/increasing-order-search-tree/)，以中序為概念。BST 樹經過中序後其最後會是排序結果。

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode() {}
 *     TreeNode(int val) { this.val = val; }
 *     TreeNode(int val, TreeNode left, TreeNode right) {
 *         this.val = val;
 *         this.left = left;
 *         this.right = right;
 *     }
 * }
 */
class Solution {
    TreeNode res = new TreeNode(0);
    TreeNode cur = res;
    public TreeNode increasingBST(TreeNode root) {
        if (root != null){
            increasingBST(root.left);
            cur.right = new TreeNode(root.val);
            cur = cur.right;
            increasingBST(root.right);
        }
        return res.right;
    }
}
```

>上述 res 是一個物件，但 cur 的記憶體是參照 res，因此 cur 的操作會影響 res。因此單純操作 res 最後結果會是最後遍歷的結果。

