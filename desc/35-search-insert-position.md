## 二分查找 target，找到了直接返回；找不到时，循环结束时 left 刚好停在第一个比 target 大的位置，就是该插入的地方。
```
class Solution {
public:
    int searchInsert(vector<int>& nums, int target) {
        int left=0;
        int right = nums.size()-1;

        while(left<=right)
        {
            int mid = left+(right-left)/2;

            if(nums[mid]<target)
            {
                left=mid+1;
            }
            else if(nums[mid]>target)
            {
                right=mid-1;
            }
            else{
                return mid;
            }
        }
        return left;
    }
};
```