---
title: '[LeetCode] 从排序数组中删除重复项'
date: 2018-04-23 01:48:43
tags:
  - LeetCode
  - algorithm
categories: LeetCode
---

原题链接: <https://leetcode-cn.com/explore/interview/card/top-interview-questions-easy/1/array/21/>

### 初级算法(数组类)

给定一个排序数组，你需要在原地删除重复出现的元素，使得每个元素只出现一次，返回移除后数组的新长度。
不要使用额外的数组空间，你必须在原地修改输入数组并在使用 O(1) 额外空间的条件下完成。

示例 1:

        给定数组 nums = [1,1,2], 
        函数应该返回新的长度 2, 并且原数组 nums 的前两个元素被修改为 1, 2。 
        你不需要考虑数组中超出新长度后面的元素。

示例 2:

        给定 nums = [0,0,1,1,1,2,2,3,3,4],
        函数应该返回新的长度 5, 并且原数组 nums 的前五个元素被修改为 0, 1, 2, 3, 4。
        你不需要考虑数组中超出新长度后面的元素。

### 我的解答(12ms)

```
int removeDuplicates(int* nums, int numsSize)
{
        int i = 0;
        int valid = 0;

        /* 兼容空数组 */
        if(numsSize == 0)
                return 0;

        for(i = 0; i < numsSize; i++)
        {
                if(nums[i] != nums[valid])
                {
                        valid++;
                        nums[valid] = nums[i];
                }
        }

        return valid + 1;
}
```

### 最优解答(8ms)

```
int removeDuplicates(int* nums, int numsSize) {
        int i = -1, j;
        if (numsSize > 0)
        {
                for (i = 0, j = 1; i < numsSize; i++)
                {
                        while (nums[i] == nums[j] && j < numsSize)
                        {
                                j++;
                        }
                        if (j == numsSize)
                        {
                                break;
                        }
                        nums[i + 1] = nums[j];
                }
        }
	return i + 1;
}
```

### 分析不足

待补充！