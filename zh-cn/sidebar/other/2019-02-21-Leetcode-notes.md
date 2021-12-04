---
layout: post
title: leetcode刷题笔记
categories: leetcode
description: leetcode刷题笔记整理，希望能够坚持下去
keywords: leetcode
---

#### 3.无重复字符的最长字串
给定一个字符串，请你找出其中不含有重复字符的 最长子串 的长度

***示例***

输入: "abcabcbb"
输出: 3 
解释: 因为无重复字符的最长子串是 "abc"，所以其长度为 3。

输入: "bbbbb"
输出: 1
解释: 因为无重复字符的最长子串是 "b"，所以其长度为 1。

输入: "pwwkew"
输出: 3
解释: 因为无重复字符的最长子串是 "wke"，所以其长度为 3。请注意，你的答案必须是子串的长度，"pwke" 是一个子序列，不是子串。

```
class Solution {
    public int lengthOfLongestSubstring(String s) {
        
        if (s == null || s.length() <=0) return 0;
        
        Set<Character> set = new HashSet<>();
        int res =0;
        int left = 0;
        int right = 0;
        
        while(right < s.length()){
            if (!set.contains(s.charAt(right))){
                set.add(s.charAt(right++));
                res = Math.max(res, set.size());
            }else{
                set.remove(s.charAt(left++));
            }
        }
        
        return res;
    }
}
```

#### 2.两数相加
给出两个 非空 的链表用来表示两个非负的整数。其中，它们各自的位数是按照 逆序的方式存储的，并且它们的每个节点只能存储一位数字。如果，我们将这两个数相加起来，则会返回一个新的链表来表示它们的和。您可以假设除了数字 0 之外，这两个数都不会以 0 开头。

***示例***

输入：(2 -> 4 -> 3) + (5 -> 6 -> 4)
输出：7 -> 0 -> 8
原因：342 + 465 = 807

```
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        ListNode head = new ListNode(0);
        ListNode node = head;
        ListNode p1 = l1;
        ListNode p2 = l2;
        int carry = 0;
        while(p1 != null || p2 != null){
            int x = p1 == null?0:p1.val;
            int y = p2 == null?0:p2.val;
            int sum = x+y+carry;
            carry = sum/10;
            ListNode next = new ListNode(sum%10);
            node.next = next;
            node = node.next;
            if(p1 != null){
                p1 = p1.next;
            }
            if(p2 != null){
                p2 = p2.next;
            }
        }
        if (carry > 0) {
            node.next = new ListNode(carry);
        }
        return head.next;
    }
}
```

#### 1.两数之和

给定一个整数数组 nums和一个目标值 target，请你在该数组中找出和为目标值的那 两个整数，并返回他们的数组下标。
你可以假设每种输入只会对应一个答案。但是，你不能重复利用这个数组中同样的元素。

***示例***

给定 nums = [2, 7, 11, 15], target = 9
因为 nums[0] + nums[1] = 2 + 7 = 9
所以返回 [0, 1]

```
class Solution {
    public int[] twoSum(int[] nums, int target) {
        Map<Integer, Integer> map = new HashMap<>();
        int[] res = new int[2];
        for (int i=0; i<nums.length; i++){
            if (map.containsKey(target-nums[i])){
                res[0] = i;
                res[1] = map.get(target-nums[i]);
            }else{
                map.put(nums[i], i);
            }
        }
        
        return res;
    }
}
```

