# 739.每日温度

# 题目描述

给定一个整数数组 `temperatures` ，表示每天的温度，返回一个数组 `answer` ，其中 `answer[i]` 是指对于第 `i` 天，下一个更高温度出现在几天后。如果气温在这之后都不会升高，请在该位置用 `0` 来代替。

**示例 1:**

```
输入: temperatures = [73,74,75,71,69,72,76,73]
输出: [1,1,4,2,1,1,0,0]
```

**示例 2:**

```
输入: temperatures = [30,40,50,60]
输出: [1,1,1,0]
```

**示例 3:**

```
输入: temperatures = [30,60,90]
输出: [1,1,0]
```

**提示：**

- `1 <= temperatures.length <= 105`
- `30 <= temperatures[i] <= 100`

---
## 单调递减栈存索引，当前温度比栈顶高时，栈顶那天就等到了答案，计算天数差后弹出；当前索引入栈等后面更高的温度来"解决"。

```
class Solution {
public:
    vector<int> dailyTemperatures(vector<int>& temperatures) {
        int n=temperatures.size();
        vector<int> res(n,0);
        stack<int> sk;

        for(int i=0;i<n;i++)
        {
            while(!sk.empty() && temperatures[i]>temperatures[sk.top()])
            {
                int prevDay = sk.top();
                res[prevDay] = i-prevDay;
                sk.pop();
            }
            sk.push(i);
        }
        return res;
    }
};
```