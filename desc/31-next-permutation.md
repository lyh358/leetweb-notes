### 从后往前找第一个"升序对"（左边小的那个），再从后面找个比它大的最小数交换，最后把后面反转成升序。 如果全程降序，说明已经是最大排列，直接全部反转成最小。

```
class Solution {
public:
    void nextPermutation(vector<int>& nums) {
        int n = nums.size();
         
        if(n<=1) return;
        //找i
        int i = n-2;
        while(i>=0 && nums[i]>=nums[i+1])
        {
            i--;
        }
        //找j
        int j=n-1;
        //走这步就是没到最大
        if(i>=0)
        {
            while(j>i && nums[j]<=nums[i])
            {
                j--;
            }
            swap(nums[i],nums[j]);
        }
        //本身就是最大排列，i=-1，直接反转整串得到最小
        //反转
        reverse(nums.begin()+i+1,nums.end());
    }
};
```