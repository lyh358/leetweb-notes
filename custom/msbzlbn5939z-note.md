# 核心思路：滑动窗口



# 完整代码

```c++
class Solution {
public:
    int maxScore(vector<int>& cardPoints, int k) 
    {
        
        int max_ans=0;
        
        int len = cardPoints.size();

        //先令最大值max_ans初始化为最左边k个
        for(int i = 0;i < k;i++)
        {
            max_ans += cardPoints[i];
        }

        //传递给当前值cur_ans
        int cur_ans = max_ans;

        //每次从左边的（靠右）拿一个到右边
        for(int i = 1;i <= k;i++)
        {
            //动态平衡当前的总值，在初始基础上，把左边拿走的减去，把右边新拿的加上
            cur_ans += cardPoints[len-i]-cardPoints[k-i];
            //更新最大值（当前？历史？）
            max_ans = max(max_ans,cur_ans);
        }
        //输出最大值
        return max_ans;
    }
};
```

