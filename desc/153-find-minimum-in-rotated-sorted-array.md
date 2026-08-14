
### 旋转数组二分：哪一半有序，就用它的左端点更新最小值答案，然后去另一半（无序段）继续找。 有序段的最小值就在端点，无序段里藏着真正的最小值。
```
class Solution {
public:
    int findMin(vector<int>& nums) {
        //以最小值为分界线，其左右两边都是单调递增的数组
        int ans=1e9;
        int l=0,r=nums.size()-1;
        while(l<=r)
        {
            int mid=l+(r-l)/2;
            //如果左边是连续单调增的
            if(nums[l]<=nums[mid])
            {
                //更新最小值
                ans=min(ans,nums[l]);
                //更新后舍弃左边
                l=mid+1;
            }
            //如果右边是连续单调增
            else
            {
                ans=min(ans,nums[mid]);
                r=mid-1;
            }
        }
        return ans;
    }
};
```