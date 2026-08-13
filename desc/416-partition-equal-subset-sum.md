# 416. 分割等和子集

给你一个 只包含正整数 的 非空 数组 nums 。请你判断是否可以将这个数组分割成两个子集，使得两个子集的元素和相等。

 

**示例 1：**
```
输入：nums = [1,5,11,5]
输出：true
解释：数组可以分割成 [1, 5, 5] 和 [11] 。
```
**示例 2：**
```
输入：nums = [1,2,3,5]
输出：false
解释：数组不能分割成两个元素和相等的子集。
 ```

**提示：**
```
1 <= nums.length <= 200
1 <= nums[i] <= 100
```
### 问题转化：总和要是奇数肯定分不成，否则目标就是从数组里挑一些数，凑出总和的一半。
dp[j] 的含义：能不能从已处理的数字中，凑出和为 j。
外层循环（遍历每个 num）：逐个考虑每个数字，决定选它还是不选它。
内层循环倒序的原因：这是 0/1 背包，每个数只能用一次。如果正序遍历 j，当更新 dp[j] 时，dp[j-num] 可能已经被同一个 num 更新过了，导致这个 num 被重复用了多次。倒序保证 dp[j-num] 还是上一轮（没选当前 num）的状态。
状态转移：
cpp
dp[j] = dp[j] || dp[j - num];
//      不选num      选num（前提是前面能凑出 j-num）
初始化：dp[0] = true，和为 0 永远能凑出（什么都不选）。
最后返回 dp[target]，看能不能凑出一半。
```
class Solution {
public:
    bool canPartition(vector<int>& nums) {
        int sum=0;
        for(int num:nums)
        {
            sum+=num;
        }
        if(sum%2!=0) return false;
        int target=sum/2;

        vector<int> dp(target+1,0);
        dp[0]=1;
        
        for(auto num:nums)
        {
            for(int j = target;j>=num;j--)
            {
                dp[j]=dp[j] || dp[j-num];
            }
        }
        return dp[target];
    }
};
```