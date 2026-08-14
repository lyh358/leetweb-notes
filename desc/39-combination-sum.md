### 每个数可以无限次用：先走"不选当前数，去下一位"的分支，再走"选当前数，target 减掉它，还是从当前位继续选"的分支，target 减到 0 就收集答案。
```
class Solution {
public:
    vector<int> combination;
    vector<vector<int>> ans;
    void dfs(vector<int>& candidates,int target,int i)
    {
        if(i==candidates.size())
        {
            return;
        }
        if(target==0)
        {
            ans.push_back(combination);
            return;
        }
        dfs(candidates,target,i+1);

        if(candidates[i]<=target)
        {
            combination.push_back(candidates[i]);
            dfs(candidates,target-candidates[i],i);
            combination.pop_back();
        }
    }
    vector<vector<int>> combinationSum(vector<int>& candidates, int target) {
        dfs(candidates,target,0);
        return ans;
    }
};
```