---
title: LeetCode 53. Maximum Subarray
date: 2020-12-08
description: "divide and conquer"
tags: [LeetCode]
draft: false
---

[題目](https://leetcode.com/problems/maximum-subarray/)，求連續最大子序列。這題配合著此[youtube](https://www.youtube.com/watch?v=aTAH1Fer8xU&ab_channel=VivianNTUMiuLab)實作。透過各個擊破方式，增加程式運行效率。同時在進行切分時，有三種方案

1. 左邊最大
2. 右邊最大
3. 中間往左和往右

其第三種會有同時跨佐和跨右問題，其解決辦法如下

```java
class Solution {
    public int maxSubArray(int[] nums) {
        
        return maxSubArray(nums, 0, nums.length-1);
    }
    /**
     * case 1. left Max
     * case 2. right Max
     * case 3. mid Max
    **/
    public int maxSubArray(int[] nums, int start, int end){
        if(start == end){
            return nums[start];
        }   
        int mid = (start + end)/2;
        return Math.max(
            Math.max(maxSubArray(nums, start, mid), maxSubArray(nums, mid+1, end)),
            MaxSubCrossArray(nums, start, mid, end)
        );
        
    }
    /**
     * 中間往左和往右計算連續最大值，最後在與往左和往右相加進行比較
    **/
    public int MaxSubCrossArray(int[] nums, int start, int mid, int end){
        int sum = 0;
        int left_sum = Integer.MIN_VALUE;
        for(int i = mid; i >= start; i--){
            sum += nums[i];
            if(sum > left_sum){
                left_sum = sum;
            }
        }
        sum = 0;
        int right_sum = Integer.MIN_VALUE;
        for(int j = mid + 1; j <= end; j++){
            sum += nums[j];
            if(sum > right_sum){
                right_sum = sum;
            }
        }
        return Math.max(left_sum + right_sum, Math.max(left_sum, right_sum));
    }
}
```