---
title: 783. Minimum Distance Between BST Nodes
date: 2020-12-09
description: "in-Order Traversal"
tags: [LeetCode]
draft: false
---

[題目](https://leetcode.com/problems/minimum-distance-between-bst-nodes/)，找出兩節點最小差值，思路以中序概念來想，經過 BST 樹經過中序遍歷後其結果會是排序的樣子，因此我們只要當前的值和左邊的值進行相減逐一比較即可獲得答案。


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
    int res = Integer.MAX_VALUE;
    TreeNode node = null;
    public int minDiffInBST(TreeNode root) {
        if (root == null) return 0;
        minDiffInBST(root.left);
        if (node != null){
            System.out.println(root.val +"  "+ node.val);
            res = Math.min(res, root.val - node.val);
        }
        node = root;
        minDiffInBST(root.right);
        return res;
    }
}
```