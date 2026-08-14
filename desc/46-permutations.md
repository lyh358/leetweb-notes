# 46.全排列

# 题目描述

给定一个不含重复数字的数组 `nums` ，返回其 *所有可能的全排列* 。你可以 **按任意顺序** 返回答案。

**示例 1：**

```
输入：nums = [1,2,3]
输出：[[1,2,3],[1,3,2],[2,1,3],[2,3,1],[3,1,2],[3,2,1]]
```

**示例 2：**

```
输入：nums = [0,1]
输出：[[0,1],[1,0]]
```

**示例 3：**

```
输入：nums = [1]
输出：[[1]]
```

**提示：**

- `1 <= nums.length <= 6`
- `-10 <= nums[i] <= 10`
- `nums` 中的所有整数 **互不相同**

---
### 回溯填位置：first 是当前要填的坑，把后面每个数依次 swap 到当前位，递归填下一个坑，填完 swap 回来恢复原状。所有坑填完就是一个排列，收集答案。
```
class Solution {
public:
    void backtracing(vector<vector<int>>& ans,int first,vector<int>& nums)
    {
        if(first==nums.size())
        {
            ans.push_back(nums);
            return;
        }
        for(int i=first;i<nums.size();i++)
        {
            swap(nums[first],nums[i]);
            backtracing(ans,first+1,nums);
            swap(nums[first],nums[i]);
        }
    }
    vector<vector<int>> permute(vector<int>& nums) {
        vector<vector<int>> ans;
        backtracing(ans,0,nums);
        return ans;
    }
};
```