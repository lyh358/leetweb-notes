```
class Solution {
public:
    int maxProfit(vector<int>& prices) {
        int minBuy = INT_MAX;
        int maxSell = 0;

        for(auto price:prices)
        {
            minBuy = min(minBuy,price);
            maxSell = max(maxSell,price-minBuy);
        }
        return maxSell>0? maxSell:0;
    }
};
```