```cpp
class Solution {
public:
    bool find132pattern(vector<int>& nums) {
        stack<int> st;                    // 单调递减栈，存"3"的候选（大数）
        int n = nums.size(), k = INT_MIN;  // k 就是"2"，初始化为极小值
        for(int i = n - 1; i >= 0; i--){   // 从右往左遍历，找"1"
            if(nums[i] < k) return true;   // 找到 1 < 2，132模式成立！
            while(!st.empty() and st.top() < nums[i]) {  // 当前比栈顶大，形成"峰"
                k = max(k, st.top());      // 更新"2"为更大的被弹出值
                st.pop();                  // 弹出较小的栈顶
            }
            st.push(nums[i]);              // 当前入栈，作为后面元素的"3"候选
        }
        return false;/// 遍历完没找到
    }
};
```
