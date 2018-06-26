
---
title: leetcode 两数之和/两数相加
categories:
- 算法、数据
tags:
- leetcode
---
## 概要
前端对算法的需求一直是最少的，接触的无非是一些冒泡排序之类的简单算法，最近面试感触良多，不说面试算法已经是必考的题目，就说能够加深对Js的理解，也是要仔细看下
立个flag，对leetcode Js刷题，估计之后会再用GoLang走一遍

### 两数之和

给定一个整数数组和一个目标值，找出数组中和为目标值的两个数。

你可以假设每个输入只对应一种答案，且同样的元素不能被重复利用。
示例：
```
给定 nums = [2, 7, 11, 15], target = 9

因为 nums[0] + nums[1] = 2 + 7 = 9
所以返回 [0, 1]
```
#### 直观暴力解法，双重循环
耗时：88ms
```
/**
 * @param {number[]} nums
 * @param {number} target
 * @return {number[]}
 */
var twoSum = function(nums, target) {
    for (var i = 0; i < nums.length; i++) {
        for (var j = i+1; j < nums.length; j++) {
            if (nums[i] + nums[j] === target) {
                return [i, j]    
            }
        }
    }
};
```
这也是我的第一种解法，明显不符合面试官的要求，这个算法的时间复杂度是O(n^2)。那么只能想个O(n)的算法来实现，

#### 字典查找
整个实现步骤为：先遍历一遍数组，建立map数据，然后再遍历一遍，开始查找，找到则记录index。代码如下：
耗时：88ms
```
var twoSum = function(nums, target) {
    let m = {};
    for (var i = 0; i < nums.length; i++) {
        let tmp = target - nums[i]
        if (m[tmp] !== undefined) {
            return [m[tmp], i]
        }
        m[nums[i]] = i
    }
};
```


### 两数相加

给定两个非空链表来表示两个非负整数。位数按照逆序方式存储，它们的每个节点只存储单个数字。将两数相加返回一个新的链表。

你可以假设除了数字 0 之外，这两个数字都不会以零开头。
示例：

```
输入：(2 -> 4 -> 3) + (5 -> 6 -> 4)
输出：7 -> 0 -> 8
原因：342 + 465 = 807
```

这道题主要考察连接结构，对链表结构并不熟悉，所以想起来有问题，下面的解法有用的位运算符等
```
var addTwoNumbers = function(l1, l2) {
  var add = 0
    , ans
    , head;

  while(l1 || l2) {
    var a = l1 ? l1.val : 0
      , b = l2 ? l2.val : 0;

    var sum = a + b + add;
    add = ~~(sum / 10);

    var node = new ListNode(sum % 10);

    if (!ans)
      ans = head = node;
    else {
      head.next = node;
      head = node;
    }

    if (l1)
      l1 = l1.next;
    if (l2)
      l2 = l2.next;
  }

  if (add) {
    var node = new ListNode(add);
    head.next = node;
    head = node;
  }

  return ans;
};
```
