---
title: LeetCode 2. Add Two Numbers
date: 2020-12-13
description: "Linked List"
tags: [LeetCode, Linked List]
draft: false
---

[題目 2. Add Two Numbers](https://leetcode.com/problems/add-two-numbers/)，單純的將兩個鏈接做相加。須注意的是進位而已。


```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode() {}
 *     ListNode(int val) { this.val = val; }
 *     ListNode(int val, ListNode next) { this.val = val; this.next = next; }
 * }
 */
class Solution {
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        int carry = 0;
        ListNode dummy = new ListNode(0);
        ListNode cur = dummy;
        int l1Val = 0;
        int l2Val = 0;
        int sum = 0;
        while (l1 != null || l2 != null){
            if (l1 == null) {
                l1Val = 0;
            } else {
                l1Val = l1.val;
            }
            if (l2 == null) {
                l2Val = 0;
            } else {
                l2Val = l2.val;
            }
            sum = carry + l1Val + l2Val;
            carry = sum / 10;
            cur.next = new ListNode(sum%10);
            cur = cur.next;
            if (l1 != null){
                l1 = l1.next;  
            } 
            if (l2 != null){
                l2 = l2.next;  
            } 
            
        }
        if (carry == 1){
            cur.next = new ListNode(1);
        }
        
        return dummy.next;
    }
}
```

