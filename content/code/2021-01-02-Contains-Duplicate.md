---
title: LeetCode Contains Duplicate I and II
date: 2021-01-02
description: "array"
tags: [JAVA, LeetCode]
draft: false
---

## Contains Duplicate
[題目 217](https://leetcode.com/problems/contains-duplicate/)，檢查數組中是否有相同的值。思路一使用 Hash 方式，如果出現相同值 Hash 值會撞，如果撞表示是重複值。思路二使用 Set 不存放相同的值，最後比較 Set 存放的元素個數使否小於原始數組元素個數。

```java
class Solution {
    public boolean containsDuplicate(int[] nums) {
        HashMap<Integer, Integer> map = new HashMap<>();
        for(int i=0; i<nums.length; i++){
            if (map.containsKey(nums[i])){
                return true;
            } else {
                map.put(nums[i], 1);
            }
        }
        return false;
    }
}
```

```java
class Solution {
    public boolean containsDuplicate(int[] nums) {
        Set<Integer> set = Arrays.stream(nums).boxed().collect(Collectors.toSet());
        return set.size() < nums.length;
    }
}
```

## Contains Duplicate II

[題目 219](https://leetcode.com/problems/contains-duplicate-ii/)，相較於第一版則是多了一個條件，把出現重複的值其所在的元素位置進行相減，並且要小於或等於給定的 k。這邊延伸第一版的程式碼，只是多了位置的判別。


```java
class Solution {
    public boolean containsNearbyDuplicate(int[] nums, int k) {
        Map<Integer, Integer> map = new HashMap<>();
        for (int i=0; i<nums.length; i++){
            if(map.containsKey(nums[i])){
                if ((i - map.get(nums[i])) <= k){
                    return true;
                }
            }
            map.put(nums[i], i);
        }
        return false;
    }
}
```