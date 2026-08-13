思考已完成
假设第一个数是候选，相同+1票，不同-1票，票数归零就换人。 多数元素超过一半，两两抵消后它一定剩到最后。
```
class Solution {
public:
    int majorityElement(vector<int>& nums) {
        int candidate = nums[0];
        int vote = 1;

        for(int i=1;i<nums.size();i++)
        {
            if(vote>0)
            {
                if(nums[i]==candidate)
                {
                    vote++;
                }
                else
                {
                    vote--;
                }
            }
            else
            {
                candidate=nums[i];
                vote=1;
            }
        }
        return candidate;
    }
};
```