```
class Solution {
public:
    int canCompleteCircuit(vector<int>& gas, vector<int>& cost) {
        int n = gas.size();
        int total = 0;
        int curOil = 0;
        int start = 0;

        for(int i = 0; i < n; i++)
        {
            int diff = gas[i] - cost[i];
            total += diff;
            curOil += diff;

            if(curOil < 0)
            {
                start = i + 1;
                curOil = 0;
            }
        }
        if(total >= 0)
            return start;
        else
            return -1;
    }
};
```
