```cpp
class Solution {
public:
    vector<vector> merge(vector<vector>& intervals) {
        vector<vector> merged;
        sort(intervals.begin(),intervals.end());

        for(int i=0;i<intervals.size();i++)
        {
            int left = intervals[i][0];
            int right = intervals[i][1];

            while(i+1<intervals.size() && right>=intervals[i+1][0])
            {
                right = max(right,intervals[i+1][1]);
                i++;
            }
            merged.push_back({left,right});
        }
        return merged;
    }
};
```

---

 排序+
