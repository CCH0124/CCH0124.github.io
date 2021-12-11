---
title: LeetCode 268 Missing Number
date: 2021-01-12
description: "array"
tags: [LeetCode]
draft: false
---


## Missing Number

[題目](https://leetcode.com/problems/missing-number/) 找出陣列中缺少的數值。思路利用數組長度帶入梯形公式算出3整個序列之和再減去目前數組中所有數值之和，即可獲取答案。

```java
class Solution {
    public int missingNumber(int[] nums) {
        int len = nums.length;
        int trape = (0 + len)*(len+1)/2;
        int sum = Arrays.stream(nums).sum();
        return trape - sum;
    }
}
```
