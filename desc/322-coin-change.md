# 力扣 322. 零钱兑换 — 学习笔记（踩坑版）

> **题目**：给你一个整数数组 `coins`，表示不同面额的硬币；以及一个整数 `amount`，表示总金额。计算并返回可以凑成总金额所需的 **最少的硬币个数**。如果没有任何一种硬币组合能组成总金额，返回 `-1`。  
> **难度**：中等  
> **标签**：动态规划、完全背包

---
```


## 一、最终通过的代码

```cpp
class Solution {
public:
    int coinChange(vector<int>& coins, int amount) {
        // dp[i] = 组成 i 元所需的最少硬币数
        // 用极大值填充，表示"初始假设凑不出"
        // ❗ 注意：不用 INT_MAX，因为后面 +1 会 int 溢出！
        vector<int> dp(amount + 1, 1e6);

        // 边界条件：凑出 0 元需要 0 枚硬币
        dp[0] = 0;

        // 遍历每一个目标金额，计算其最少硬币数
        for (int i = 1; i <= amount; i++) {
            // 对于每个金额 i，遍历所有硬币面额，找最优的前一个状态
            for (int j = 0; j < coins.size(); j++) {

                // ❗ 关键判断：只有硬币面额不超过当前金额，才能进行状态转移
                // 否则 i - coins[j] 为负数，数组越界！
                if (coins[j] <= i) {

                    // 状态转移：
                    // 上一个状态 dp[i - coins[j]] 凑出了 i-coins[j] 元
                    // 再加上当前这枚硬币（+1），就能凑出 i 元
                    // 取所有硬币中的最小值
                    dp[i] = min(dp[i], dp[i - coins[j]] + 1);
                }
            }
        }

        // 判断结果：如果 dp[amount] 还是初始极大值，说明凑不出来
        // 技巧：如果全用 1 元硬币，最多只需要 amount 枚
        // 所以如果 dp[amount] > amount，说明连全用 1 元都做不到，即凑不出
        return dp[amount] > amount ? -1 : dp[amount];
    }
};
```

---

