```
class Solution {
public:
    vector<vector<int>> intervalIntersection(vector<vector<int>>& firstList, vector<vector<int>>& secondList) {
        vector<vector<int>> res;
        int i = 0, j = 0;
        int m = firstList.size();
        int n = secondList.size();

        while(i < m && j < n)
        {
            int a1 = firstList[i][0];
            int a2 = firstList[i][1];
            int b1 = secondList[j][0];
            int b2 = secondList[j][1];

            int s = max(a1, b1);
            int e = min(a2, b2);
            if(s <= e)
            {
                res.push_back({s, e});
            }

            // 区间终点小的那个指针移动
            if(a2 < b2)
                i++;
            else
                j++;
        }
        return res;
    }
};
```
