```
class Solution {
public:
    int trap(vector<int>& height) {
        int left=0,right=height.size()-1;
        int left_heightest=0,right_heightest=0;
        int ans=0;

        while(left<right)
        {
            //左右指针哪边更矮哪边走，趋向于找到一个“最低中心”
            if(height[left]<height[right])
            {
                //只有当当前指针比之前的最高边矮，才能存水
                if(height[left]<left_heightest)
                {   
                    //存的水就是高度差*1=高度差
                    ans+=left_heightest-height[left];
                }
                else
                {
                    //存不了水则更新最高边界
                    left_heightest=height[left];
                }
                //向中心逼近
                left++;
            }
            else
            {
                if(height[right]<right_heightest)
                {
                    ans+=right_heightest-height[right];
                }
                else
                {
                    right_heightest=height[right];
                }
                right--;
            }
        }
        return ans;
    }
};
```