哈希表+qianzh
```
class Solution {
public:
    int subarraySum(vector<int>& nums, int k) {
        unordered_map<int,int> countPrefix;     // <前缀和, 出现次数>
        countPrefix[0]=1;       //  crucial！处理从索引0开始的子数组
        
        int ans=0;      // 答案计数
        int sum=0;      // 当前前缀和

        for(auto num:nums)
        {
            sum+=num;   // 更新当前前缀和

            // 查前面有多少个 prefix = sum - k
            // 这些位置都能和当前位置构成和为k的子数组
            ans+=countPrefix[sum-k];    //当前sum，sum-(sum-k)==k
            
            // 把当前前缀和加入哈希表，供后面使用
            countPrefix[sum]++;
        }
        return ans;
    }
};
```