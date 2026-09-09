```python

```

```
class Solution {
public:
    int maxScore(vector<int>& cardPoints, int k) {
        int best = 0;
        int sum = 0;
        int n = cardPoints.size();

        //先全拿右边，然后用这个值初始化best
        for(int i=0;i<k;i++)
        {
            sum += cardPoints[n-1-i];
        }
        best = sum;

        //然后循环每次右边去掉一个，左边增加一个，更新sum，与best对比更新最大值
        for(int i=0;i<k;i++)
        {
            sum = sum + cardPoints[i] - cardPoints[n-k+i];//要定义变量，不能直接比较右值

            best = max(best,sum);
        }
        return best;
    }
};
```
