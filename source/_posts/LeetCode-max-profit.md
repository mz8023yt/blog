---
title: '[LeetCode] 买卖股票的最佳时机 II'
date: 2018-04-23 02:21:57
tags:
  - LeetCode
  - algorithm
categories: LeetCode
---

原题链接: <https://leetcode-cn.com/explore/interview/card/top-interview-questions-easy/1/array/22/>

### 初级算法(数组类)

给定一个数组，它的第 i 个元素是一支给定股票第 i 天的价格。
设计一个算法来计算你所能获取的最大利润。你可以尽可能地完成更多的交易（多次买卖一支股票）。

> 注意：你不能同时参与多笔交易（你必须在再次购买前出售掉之前的股票）。

示例 1:

        输入: [7,1,5,3,6,4]
        输出: 7
        解释: 在第 2 天（股票价格 = 1）的时候买入，在第 3 天（股票价格 = 5）的时候卖出, 这笔交易所能获得利润 = 5-1 = 4 。
             随后，在第 4 天（股票价格 = 3）的时候买入，在第 5 天（股票价格 = 6）的时候卖出, 这笔交易所能获得利润 = 6-3 = 3 。

示例 2:

        输入: [1,2,3,4,5]
        输出: 4
        解释: 在第 1 天（股票价格 = 1）的时候买入，在第 5 天 （股票价格 = 5）的时候卖出, 这笔交易所能获得利润 = 5-1 = 4 。
             注意你不能在第 1 天和第 2 天接连购买股票，之后再将它们卖出。
             因为这样属于同时参与了多笔交易，你必须在再次购买前出售掉之前的股票。

示例 3:

        输入: [7,6,4,3,1]
        输出: 0
        解释: 在这种情况下, 没有交易完成, 所以最大利润为 0。

### 我的解答(4ms)

```
int maxProfit(int* prices, int pricesSize)
{
        int i = 0;

        /* 表征是否买入的状态标志位，尚未买入 status = 0，已经买入 status = 0 */
        int status = 0;

        /* 其中一次买获得的收益 */
        int profit = 0;

        /* 所有买获得的总收益 */
        int total = 0;

        for(i = 0;  i < pricesSize - 1; i++)
        {
                /* 如果后一天价格更高，并且没有买入，那赶紧买起来 */
                if((prices[i + 1] > prices[i]) && (status == 0))
                {
                        status = 1;
                        profit = prices[i];
                }

                /* 如果后一天价格跌了，股票还在手上，那赶紧卖了呀 */
                if ((prices[i + 1] < prices[i]) && (status == 1)) 
                {
                        status = 0;
                        profit = prices[i] - profit;
                        total = total + profit;
                }
        }

        /* 最后一天股票还在手上，赶紧卖 */
        if(status == 1)
        {
                profit = prices[i] - profit;
                total = total + profit;
        }

        return total;
}
```

### 最优解答(0ms)

```
int maxProfit(int prices[], int n)
{
        int profit = 0;
        for(int i = 0; i < n - 1; i++)
        {
                int temp = prices[i+1] - prices[i];
                if(temp > 0)
                profit += temp;
        }
        return profit;
}
```

### 分析不足

待补充！
