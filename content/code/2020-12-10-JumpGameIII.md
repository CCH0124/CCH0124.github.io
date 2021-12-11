---
title: LeetCode 1306. Jump Game III
date: 2020-12-10
description: "深度優先搜尋"
tags: [LeetCode, recursion, dfs]
draft: false
---

[題目](https://leetcode.com/problems/jump-game-iii/)，會給一個起始點(start)，而這個起始點可以選擇對元素進行加或減，透過這種方式不斷進行，最後檢測有沒有辦法到達值 0 的元素。

思路就是要麻當前元素進行減或加，用遞迴方式不斷去求值。使用一個 `boolean` 的陣列記錄當前該位置是否訪問過。

```java
class Solution {
    
    public boolean canReach(int[] arr, int start) {
        boolean [] vis = new boolean[arr.length];
        return canReach(arr, start, vis);
    }
    private boolean canReach(int[] arr, int start, boolean[] vis){
        if (start < arr.length && start >= 0 && !vis[start]){
            if (arr[start] == 0){
                return true;
            }
            vis[start] = true;
            return canReach(arr, start + arr[start], vis) || canReach(arr, start - arr[start], vis);
        }
        
        return false;
    }
}
```
