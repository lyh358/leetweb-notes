**一次遍历，每个数先当成 2 放到末尾；如果是 1 或 0，就在 p1 位置补个 1；如果还是 0，再在 p0 位置补个 0。 三个指针各管各的，0 把 1 往后挤，1 把 2 往后挤，一遍搞定。**
```
class Solution {
public:
    void sortColors(vector<int>& nums) {
        int p0 = 0;
        int p1 = 0;

        for(int i=0;i<nums.size();i++)
        {
            int temp = nums[i];

            nums[i]=2;
            if(temp==1 || temp==0)
            {
                nums[p1]=1;
                p1++;
            }
            if(temp==0)
            {
                nums[p0]=0;
                p0++;
            }
        }
    }
};
```