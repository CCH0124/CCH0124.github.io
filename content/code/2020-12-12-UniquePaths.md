---
title: LeetCode 62. Unique Paths and 63. Unique Paths II
date: 2020-12-12
description: "dynamic programming"
tags: [LeetCode, dp]
draft: false
---

[題目 62](https://leetcode.com/problems/unique-paths/) 與 [題目 63](https://leetcode.com/problems/unique-paths-ii/) 是相似的問題，後者多了障礙物的設計。思路就是利用動態規劃，其中當前位置會是左邊與上面的能夠抵達方式的相加，因為題目限制往右和往下走。

題目 62

```java
class Solution {
    public int uniquePaths(int m, int n) {
        int [][] dp =  new int[m][n];
        for(int i=0; i<m; i++){
            for(int j=0; j<n; j++){
                if( i == 0 || j == 0 ) {
                    dp[i][j] = 1;
                } else {
                    dp[i][j] = dp[i-1][j] + dp[i][j-1];
                }
            }
        }
        return dp[m-1][n-1];
    }
}
```

題目 63，相較於 62 多了很多比較。我們把障礙物的當前能夠抵達次數設為 `0`，其做相加時不影響結果。接著起點做個判斷，非障礙物將其設為 `1` 以進行後續計算，在邊緣的地方有可能存在障礙物，因此不能像題目 62 一樣直接都設為 `1`。

```java
class Solution {
    public int uniquePathsWithObstacles(int[][] obstacleGrid) {
        if (obstacleGrid[0][0] == 1) return 0;
        for(int i=0; i<obstacleGrid.length; i++){
            for(int j=0; j<obstacleGrid[0].length; j++){
                if (obstacleGrid[i][j] == 1){
                    obstacleGrid[i][j] = 0;
                } else if ( i == 0 && j == 0){
                    obstacleGrid[i][j] = 1;
                } else if (i == 0){
                    obstacleGrid[i][j] = obstacleGrid[i][j-1];
                } else if (j == 0){
                    obstacleGrid[i][j] = obstacleGrid[i-1][j];
                } else {
                    obstacleGrid[i][j] = obstacleGrid[i-1][j]+obstacleGrid[i][j-1];
                }
                
            }    
        }
        
        return obstacleGrid[obstacleGrid.length-1][obstacleGrid[0].length-1];
    }
}
```