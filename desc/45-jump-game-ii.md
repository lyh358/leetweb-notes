# LeetCode 45. 跳跃游戏 II —— 贪心/BFS 分层学习笔记

> **题目**: [45. Jump Game II](https://leetcode.cn/problems/jump-game-ii/)  
> **核心**: 给定一个长度为 `n` 的 **0 索引**整数数组 `nums`。初始位置为 `nums[0]`。每个元素 `nums[i]` 表示从索引 `i` 向前跳转的最大长度。返回到达 `nums[n - 1]` 的**最小跳跃次数**。  
> **假设你总是可以到达 `nums[n - 1]`**。  
> **示例**:  
> ```
> nums = [2, 3, 1, 1, 4]  →  2（0→1→4）
> nums = [2, 3, 0, 1, 4]  →  2（0→1→4）
> ```
---
```
class Solution {
public:
    int jump(vector<int>& nums) {
        int jumptimes = 0;
        int start = 0;
        int end = 1;

        while(end<nums.size())
        {
            int maxRange = 0;
            for(int i=start;i<end;i++)
            {
                maxRange = max(maxRange, i+nums[i]);
            }
            start = end;
            end = maxRange+1;
            jumptimes++;
        }
        return jumptimes;
    }
};
```
注释版
```
class Solution {
public:
    int jump(vector<int>& nums) {
        //一直跳最远，直到跳出界，那最后一跳肯定能跳到末尾（末尾是小于出界的位置）
        int jumptimes = 0;    //最小跳跃次数
        //界定下一次起跳的范围[start,end)
        int start = 0;  //下一次起跳点范围开始的格子（初始为0）
        int end = 1;    //下一次起跳点范围结束的格子(初始为1，即第一次的范围就是一个格子)

        while(end<nums.size())  //只要不出界就一直跳
        {
            int maxPos = 0; //能挑到的最远距离
            for(int i = start;i<end;i++)    //在所有起跳范围内找能跳的最远的地方
            {
                maxPos=max(maxPos,nums[i]+i);
            }
            //更新起跳范围
            start = end;    //上一次的结束位置为本次的开始位置
            end = maxPos+1; //范围结束位置为上一轮能跳的最远位置+1（开区间）
            jumptimes++;    //跳跃次数+1
        }
        return jumptimes;
    }
};
```