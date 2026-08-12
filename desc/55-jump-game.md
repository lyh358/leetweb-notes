```
class Solution {
public:
    bool canJump(vector<int>& nums) {
        int maxRange = 0;   //能跳到的最远距离
        for(int i=0;i<nums.size();i++)  //遍历每一个位置，更新整个数组中能到达的最远距离
        {   
            if(maxRange<i) return false;    //如果当前位置不可达：返回false 
            maxRange = max(maxRange,i+nums[i]); //更新最大可达距离
        }   
        return true;    //所有位置都可达：返回true
    }
};
```