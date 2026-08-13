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