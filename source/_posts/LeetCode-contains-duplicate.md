---
title: '[LeetCode] 存在重复'
date: 2018-04-23 21:38:26
tags:
  - LeetCode
  - algorithm
categories: LeetCode
---

原题链接: <https://leetcode-cn.com/explore/interview/card/top-interview-questions-easy/1/array/24/>

### 初级算法(数组类)

给定一个整数数组，判断是否存在重复元素。
如果任何值在数组中出现至少两次，函数应该返回 true。如果每个元素都不相同，则返回 false。

### 我的解答(1780ms)

```
bool containsDuplicate(int* nums, int numsSize)
{
        if (numsSize < 2)
        {
                return false;
        }

        for(int i = (numsSize - 1); i >= 0; i--)
        {
                for(int j = i - 1; j >= 0; j--)
                {
                        if(nums[i] == nums[j])
                        {
                                return true;
                        }
                }
        }

        return false;
}
```

### 最优解答(8ms)

```
bool containsDuplicate(int* nums, int numsSize)
{
        if (numsSize < 2)
        {
                return false;
        }

        int i = 0;
        int adr = 0;
        int csh = nums[0];
        int p = numsSize;
        int *hash = (int *)malloc(numsSize * sizeof(int));

        for (i = 0; i < numsSize; i++)
        {
                csh = nums[i] > csh ? csh : nums[i];
        }

        for (i = 0; i < numsSize; i++)
        {
                hash[i] = csh - 1;
        }

        for (i = 0; i < numsSize; i++)
        {
                adr = abs(nums[i]) % p;
                while (hash[adr] == csh && hash[adr] != nums[i])
                {
                        adr = (++adr) % p;
                }
                if (hash[adr] == nums[i])
                {
                        free(hash);
                        hash = NULL;
                        return true;
                }
                hash[adr] = nums[i];
        }
        free(hash);
        hash = NULL;
        return false;
}
```

### 分析不足

待补充！
