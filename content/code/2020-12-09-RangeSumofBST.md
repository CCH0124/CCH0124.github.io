---
title: LeetCode 938. Range Sum of BST
date: 2020-12-09
description: "Pre-Order Traversal"
tags: [LeetCode]
draft: false
---

[題目](https://leetcode.com/problems/range-sum-of-bst/)，這一題會給 `low` 和 `high`，只要這棵樹的節點數值滿足大於等於 `low` 和小於等於 `high` 就將其相加並成為最後結果。思路是使用前序進行遍歷，並利用 `BST` 特性*左小右大*來決定下個節點是否要進行遞歸。

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
    int res = 0;
    public int rangeSumBST(TreeNode root, int low, int high) {

        if (root != null){
            if (root.val >= low && root.val <= high){
                res += root.val;
            }
            if (root.val >= low){
                rangeSumBST(root.left, low, high);    
            }
            if (root.val <= high){
                rangeSumBST(root.right, low, high);    
            }
               
        }
        return res;
    }
}
```

