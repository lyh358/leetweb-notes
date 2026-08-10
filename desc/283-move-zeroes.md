这道题有一个非常简洁的标准解法：
不去找0，只用右指针找非0，左指针记录重排位置
```cpp
class Solution {
public:
    void moveZeroes(vector<int>& nums) {
        int left = 0;  // left 指向"下一个非零元素应该放的位置"

        for (int right = 0; right < nums.size(); right++) {
            if (nums[right] != 0) {
                swap(nums[left], nums[right]);
                left++;
            }
        }
    }
};
```
踩坑记录：
1.right=1
这会忘记检查nums[0]导致第一个数如果是fei