这道题有一个非常简洁的标准解法：
不去找0，只用右指针找fei
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