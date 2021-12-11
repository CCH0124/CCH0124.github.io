---
title: LeetCode Pascal's Triangle I and II
date: 2020-12-26
description: "array"
tags: [LeetCode]
draft: false
---

## Pascal's Triangle

[題目](https://leetcode.com/problems/pascals-triangle/)，思路就是動態規劃的概念。

```java
class Solution {
    public List<List<Integer>> generate(int numRows) {
        List<List<Integer>> res = new ArrayList<>();
        if (numRows == 0) return res;
        List<Integer> base = new ArrayList<>();
        base.add(1);
        res.add(base);
        
        for (int i=1; i<numRows; i++){
            List<Integer> row = new ArrayList<>();
            row.add(1);
            for(int j=1; j < i; j++){
                row.add(res.get(i-1).get(j) + res.get(i-1).get(j-1));
            }
            row.add(1);
            res.add(row);
        }
        
        return res;
    }
}
```
## Pascal's Triangle II
[題目](https://leetcode.com/problems/pascals-triangle-ii/)相較於第一版本它是要取出某一列，概念上是相同。


```java
class Solution {
    public List<Integer> getRow(int rowIndex) {
        List<List<Integer>> res = new ArrayList<>();
        List<Integer> base = new ArrayList<>();
        base.add(1);
        res.add(base);
        if (rowIndex == 0) return res.get(0);
        for (int i=1; i<=rowIndex; i++){
            List<Integer> row = new ArrayList<>();
            row.add(1);
            for(int j=1; j < i; j++){
                row.add(res.get(i-1).get(j) + res.get(i-1).get(j-1));
            }
            row.add(1);
            res.add(row);
        }
        
        return res.get(rowIndex);
    }
}
```

遞迴方式：

```java
class Solution {
    public List<Integer> getRow(int rowIndex) {
        if(rowIndex == 0) { 
            return new ArrayList<>(Arrays.asList(1));
        } else if(rowIndex == 1) {
            return new ArrayList<>(Arrays.asList(1, 1));
        }

        return getRow(Arrays.asList(1, 1), 2, rowIndex);
    }
    
    public List<Integer> getRow(List<Integer> prevRow, int current, int rowIndex) {

        List<Integer> row = new ArrayList<>();
        row.add(1);

        for(int i = 1; i < current; i++) {
            row.add(prevRow.get(i - 1) + prevRow.get(i));
        }

        row.add(1);

        if(current == rowIndex) {
            return row;
        } else {
            return getRow(row, current + 1, rowIndex);
        }
    }
}
```