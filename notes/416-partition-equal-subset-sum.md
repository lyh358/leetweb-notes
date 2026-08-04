```md
# LeetCode 416. 分割等和子集

## 题意

给一个只包含正整数的数组 `nums`，判断能不能把它分成两个子集，使两个子集的元素和相等。

等价于：

> 能不能从数组中选一些数，让它们的和等于总和的一半。

---

## 核心思路：01 背包

假设数组总和为 `sum`。

如果能分成两个和相等的子集，那么每个子集的和一定是：

```cpp
target = sum / 2
```

所以问题变成：

> 从 `nums` 中每个数最多选一次，能不能凑出 `target`。

这就是典型的 **01 背包可行性问题**。

---

## 关键判断

如果 `sum` 是奇数，直接返回 `false`。

因为奇数不可能平均分成两个整数和。

```cpp
if (sum % 2 == 1) return false;
```

---

## dp 定义

```cpp
dp[j]
```

表示：

> 是否可以从前面若干个数中，凑出和为 `j`。

初始化：

```cpp
dp[0] = true;
```

含义是：什么都不选，可以凑出 0。

---

## 状态转移

遍历每个数 `num`，尝试把它放进背包：

```cpp
dp[j] = dp[j] || dp[j - num];
```

含义：

- `dp[j]` 原来就能凑出来
- 或者之前能凑出 `j - num`，现在加上 `num`，就能凑出 `j`

注意 `j` 必须 **倒序遍历**：

```cpp
for (int j = target; j >= num; j--)
```

因为每个数只能用一次。

如果正序遍历，就可能同一个数被重复使用，变成完全背包了。

---

## C++ 代码

class Solution {
public:
    bool canPartition(vector<int>& nums) {
        int sum = 0;
        for (int x : nums) {
            sum += x;
        }

        if (sum % 2 == 1) {
            return false;
        }

        int target = sum / 2;
        vector<bool> dp(target + 1, false);
        dp[0] = true;

        for (int num : nums) {
            for (int j = target; j >= num; j--) {
                dp[j] = dp[j] || dp[j - num];
            }
        }

        return dp[target];
    }
};
```

---

## 复杂度

```cpp
时间复杂度：O(n * target)
空间复杂度：O(target)
```

其中 `target = sum / 2`。

---

## 复习口诀

```md
分割等和子集：
先求总和，奇数直接 false；
目标变成凑 sum / 2；
每个数只能用一次，所以是 01 背包；
dp[j] 表示能否凑出 j；
倒序遍历容量，避免重复使用同一个数。
```
```