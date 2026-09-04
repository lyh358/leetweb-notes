```cpp
class Solution {public:    int maxProfit(vector<int>& prices) {
        int max_sell=0;
        if(prices.size()<2)  return 0;
        //简单的贪婪：只要今天卖出有收益，那就卖一次
        for(int i = 1;i<prices.size();i++)
        {
            if(prices[i]-prices[i-1]>0)
            {
                max_sell+=prices[i]-prices[i-1];
            }
        }
        return max_sell;
    }
};
```
