---
title: LeetCode Best Time to Buy and Sell Stock I and II
date: 2020-12-31
description: "array"
tags: [LeetCode]
draft: false
---

## Best Time to Buy and Sell Stock I
[題目](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/)說明，如果只允許最多完成一筆交易，找到最大的利潤。也就是找到最大獲利的價差。

```java
class Solution {
    public int maxProfit(int[] prices) {
        int min = Integer.MAX_VALUE;
        int max = 0;
        for (int i=0; i<prices.length; i++){
            int price = prices[i];
            if (price < min){
                min = price;
            } else if (price - min > max) {
                max = price - min;
            }
        }
        return max;
    }
}
```

## Best Time to Buy and Sell Stock II
[題目](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii/)說明，相較於第一版，這次是找出多筆交易中可獲利最大的價值。思路是，前後的價值為正數表示此交易有獲利，最後把這多筆的獲利相加。

```java
class Solution {
    public int maxProfit(int[] prices) {
        int maxprofit = 0;
        for (int i = 1; i < prices.length; i++) {
            if (prices[i]-prices[i-1] > 0){
                maxprofit += prices[i] - prices[i - 1];
            }
        }
        return maxprofit;
    }
}
```