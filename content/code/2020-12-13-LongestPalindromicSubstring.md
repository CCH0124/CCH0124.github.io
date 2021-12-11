---
title: LeetCode 5. Longest Palindromic Substring
date: 2020-12-13
description: "dynamic programming"
tags: [LeetCode, dp]
draft: false
---


[題目 5. Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring/)，這題一開始沒想出來解法，但是是知道回文規則。看了影片和一些參考資料才知道使用 DP 的原理。

```java
class Solution {
    public String longestPalindrome(String s) {
        if (s.length() == 1) return s;
        if (s.length() == 2 && s.charAt(0) == s.charAt(1)){
            return s;   
        }
        int start = 0;
        int end = 0;
    
        boolean [][] dp = new boolean[s.length()][s.length()];
        // 自己和自己是回文
        for (int i=0; i<dp.length; i++){
            dp[i][i] = true;
        }
        // 相鄰的字元相同也是回文
        for (int i = 0; i < s.length()-1; i++) {
            if (s.charAt(i) == s.charAt(i+1)) {
                dp[i][i+1] = true;
                start = i;
                end = i+1;
            }
        }
        for (int len = 2; len < s.length(); len++) {
            for (int i = 0; i < s.length()-len; i++) {
                int j = i + len;
                // 當比較某字元時，其該 dp 的位置的左半部會是計算好的結果
                if (s.charAt(i) == s.charAt(j) && dp[i+1][j-1]) {
                    dp[i][j] = true;
                    start = i;
                    end = len+i;
                }
            }
        } 
        
        return s.substring(start, end+1);
    }
}
```

## Ref

- [algotree.org](https://algotree.org/algorithms/dynamic_programming/longest_palindromic_substring/)
- [youtube 影片](https://www.youtube.com/watch?v=Fi5INvcmDos&t=582s&ab_channel=QuinstonPimenta)