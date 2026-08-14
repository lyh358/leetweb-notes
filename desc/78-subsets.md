### 对每个元素做"选/不选"两条路的 DFS：选就加入集合递归下一位，回溯弹出后再递归下一位（不选），走到数组末尾收集当前集合。 
```
class Solution {
public:
    vector<vector<int>> ans;
    vector<int> sets;

    //dfs回溯主函数
    void backtracing(vector<int>& nums, int curPos)
    {   
        //返回条件：当当前位置溢出
        if(curPos==nums.size())
        {
            //此时是一轮完整的set选择
            //加入结果数组,并回溯
            ans.push_back(sets);
            return;
        }
        
        //回溯主操作：对于一个集合，每一位数只有选/不选
        //1.选择当前数加入集合，然后进入下一位
        sets.push_back(nums[curPos]);
        backtracing(nums,curPos+1);
        //2.不选当前数加入集合（回溯后弹出即为不选），然后进入下一位
        sets.pop_back();
        backtracing(nums,curPos+1);
    }

    vector<vector<int>> subsets(vector<int>& nums) {
        backtracing(nums,0);
        return ans;
    }
};
```