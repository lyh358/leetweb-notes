
```
class Solution {
public:
    vector<int> partitionLabels(string s) {
        //哈希表统计每个字符出现的最远下标
        unordered_map<char,int> map;
        for(int i=0;i<s.size();i++)
        {
            map[s[i]]=i;
        }
        //答案数组
        vector<int> ans;
        //初始化统计区间左右端点，用于计算区间长度
        int start = 0;
        int end = 0;
        //遍历字符串：划分区间并统计
        for(int i=0;i<s.size();i++)
        {
            end = max(end,map[s[i]]);
            
            //当区间实际边界已经到达理论扩张边界了，说明区间外部没有相同元素了
            if(i==end)
            {
                //可以进行切片
                ans.push_back(end-start+1);
                //更新左边界 
                start=end+1;
            }
        }
        return ans;
    }
};
```