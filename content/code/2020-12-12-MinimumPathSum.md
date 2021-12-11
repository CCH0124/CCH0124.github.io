---
title: LeetCode 64. Minimum Path Sum
date: 2020-12-12
description: "dynamic programming"
tags: [LeetCode, dp]
draft: false
---

[題目 64. Minimum Path Sum](https://leetcode.com/problems/minimum-path-sum/) 尋找最少的路徑成本從起始點 (0,0) 到最右右下角，相似於題目 `62` 和 `63`。同樣的期限制是**向右**和**向下**。


```java
class Solution {
    public int minPathSum(int[][] grid) {
        for(int i=0; i<grid.length; i++){
            for(int j=0; j<grid[0].length; j++){
                if ( i == 0 && j == 0){
                    continue;
                }
                if (i == 0){
                    grid[i][j] = grid[i][j]+grid[i][j-1];
                } else if (j == 0){
                    grid[i][j] = grid[i][j]+grid[i-1][j];
                } else {
                    grid[i][j] = Math.min((grid[i][j] + grid[i][j-1]), (grid[i][j] + grid[i-1][j]));
                }
                
            }
        }    
        return grid[grid.length-1][grid[0].length-1];
    }
}
```


例子

![題目範例](https://assets.leetcode.com/uploads/2020/11/05/minpath.jpg)，透過程式計算會變成

```
1       3+1                         4+1
1+1     min(4+5, 2+5)               min(5+1, min(4+5, 2+5) + 1)
6       min(2+min(4+5, 2+5), 2+6)   min(min(5+1, min(4+5, 2+5) + 1)+1, min(2+min(4+5, 2+5), 2+6)+1)
```