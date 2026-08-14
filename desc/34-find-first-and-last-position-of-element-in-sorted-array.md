### 两次二分：第一次找左边界，找到 target 不返回，继续往左缩（right = mid-1）；第二次找右边界，找到 target 不返回，继续往右扩（left = mid+1）。分别记录最左和最右出现的位置。
```
class Solution {
public:
    vector<int> searchRange(vector<int>& nums, int target) {
        int n=nums.size();

        int first=-1,last=-1;   //找不到返回{-1,-1}
        
        //二分特化：找最左边的target
        int left=0,right=n-1;
        while(left<=right)
        {
            int mid=left+(right-left)/2;
            if(nums[mid]==target)
            {
                first=mid;  //找到之后先暂存，不退出
                right=mid-1;    //继续向左找
            }
            else if(nums[mid]<target)
            {
                left=mid+1;
            }
            else{
                right=mid-1;
            }
        }

        //二分特化：找最右边的target
        left=0,right=n-1;
        while(left<=right)
        {
            int mid=left+(right-left)/2;
            if(nums[mid]==target)
            {
                last=mid;
                left=mid+1;
            }
            else if(nums[mid]<target)
            {
                left=mid+1;
            }
            else{
                right=mid-1;
            }
        }
    return {first,last};
    }
};
```