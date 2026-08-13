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