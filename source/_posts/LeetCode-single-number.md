---
title: '[LeetCode] 只出现一次的数字'
date: 2018-04-23 22:32:32
tags:
  - LeetCode
  - algorithm
categories: LeetCode
---

原题链接: <https://leetcode-cn.com/problems/single-number/description/>

### 初级算法(数组类)

给定一个非空整数数组，除了某个元素只出现一次以外，其余每个元素均出现两次。找出那个只出现了一次的元素。

> 说明: 你的算法应该具有线性时间复杂度。 你可以不使用额外空间来实现吗？

示例 1:

        输入: [2,2,1]
        输出: 1

示例 2:

        输入: [4,1,2,1,2]
        输出: 4

### 我的解答(4ms)

```
int singleNumber(int* nums, int numsSize)
{
        int single = 0;

        for(int i = 0; i < numsSize; i++)
        {
                single = single ^ nums[i];
        }

        return single;
}
```

### 最优解答(4ms)

```
int singleNumber(int* nums, int numsSize)
{
        int single = nums[0];

        for(int i = 1; i < numsSize; i++)
        {
                single = single ^ nums[i];
        }

        return single;
}
```

### 分析不足

待补充！
