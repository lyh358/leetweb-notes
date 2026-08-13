### 哈希表统计每个数的频率，丢进大顶堆按频率排序，弹出堆顶 k 次就是答案。
```
class Solution {
public:
    vector<int> topKFrequent(vector<int>& nums, int k) {
        unordered_map<int,int> map;
        for(auto num:nums)
        {
            map[num]++;
        }

        vector<int> ans;
        priority_queue<pair<int,int>> pq;

        for(auto& [value,times]:map)
        {
            pq.push({times,value});
        }

        for(int i=0;i<k;i++)
        {
            ans.push_back(pq.top().second);
            pq.pop();
        }
        return ans;
    }
};
```