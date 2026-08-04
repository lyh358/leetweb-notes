## 二、解法对比总览

| 维度 | 解法一：DP 数组 | 解法二：滚动变量 |
|------|--------------|----------------|
| **空间复杂度** | O(n) | O(1) ⭐ |
| **时间复杂度** | O(n) | O(n) |
| **核心思想** | 显式存储所有子问题结果 | 只保留最近两个状态 |
| **适用场景** | 需要回溯路径、状态复杂 | 只关心最终结果 |

---

## 三、解法一：DP 数组（标准动态规划）

### 3.1 代码

```cpp
class Solution {
public:
    int rob(vector<int>& nums) {
        int n = nums.size();
        if (n == 0) return 0;

        // 子问题定义：
        // dp[k] = 偷前 k 个房间（即 nums[0..k-1]）能获得的最高金额
        vector<int> dp(n + 1);

        // 边界条件
        dp[0] = 0;           // 0 个房间，金额为 0
        dp[1] = nums[0];     // 1 个房间，只能偷它

        // 状态转移：对于第 k 个房间（nums[k-1]）
        // 选择 1：不偷 → 金额 = dp[k-1]
        // 选择 2：偷   → 金额 = dp[k-2] + nums[k-1]（不能偷第 k-1 个）
        for (int i = 2; i <= n; i++) {
            dp[i] = max(dp[i - 1], dp[i - 2] + nums[i - 1]);
        }

        return dp[n];
    }
};
```

### 3.2 关键点解析

| 代码 | 含义 |
|------|------|
| `vector<int> dp(n + 1)` | dp 数组长度为 n+1，`dp[0]` 表示 **0 个房间** 的边界情况 |
| `dp[0] = 0` | 没有房子可偷，金额为 0 |
| `dp[1] = nums[0]` | 只有一间房，只能偷它（对应 `nums` 索引 0）|
| `nums[i - 1]` | `dp[i]` 对应前 `i` 个房间，最后一个房间是 `nums[i-1]` |
| `dp[i] = max(dp[i-1], dp[i-2] + nums[i-1])` | **状态转移方程**：不偷 vs 偷 |

### 3.3 填表演示

以 `nums = [2, 7, 9, 3, 1]` 为例：

| i | nums[i-1] | dp[i-1] | dp[i-2] | 不偷 | 偷 | dp[i] |
|---|-----------|---------|---------|------|-----|-------|
| 0 | — | — | — | — | — | **0** |
| 1 | 2 | — | — | — | — | **2** |
| 2 | 7 | 2 | 0 | 2 | 0+7=7 | **7** |
| 3 | 9 | 7 | 2 | 7 | 2+9=11 | **11** |
| 4 | 3 | 11 | 7 | 11 | 7+3=10 | **11** |
| 5 | 1 | 11 | 11 | 11 | 11+1=12 | **12** |

结果：`dp[5] = 12`

---

## 四、解法二：滚动变量（空间优化）

### 4.1 核心洞察

观察状态转移方程：
```
dp[i] = max(dp[i-1], dp[i-2] + nums[i-1])
```

计算 `dp[i]` 时，**只需要 `dp[i-1]` 和 `dp[i-2]` 两个值**，更久之前的状态全部用不到。

因此可以用**两个变量**代替整个 dp 数组，将空间从 O(n) 降到 **O(1)**。

### 4.2 代码

```cpp
class Solution {
public:
    int rob(vector<int>& nums) {
        int lastOne = 0;   // 代表 dp[k-1]，即「上一间为止的最大金额」
        int lastTwo = 0;   // 代表 dp[k-2]，即「上两间为止的最大金额」

        for (auto num : nums) {
            // 当前房间的选择：
            // 1. 不偷 → lastOne（保持上一间为止的最大金额）
            // 2. 偷   → lastTwo + num（上上间最大 + 当前金额）
            int curMax = max(lastOne, lastTwo + num);

            // 状态滚动：
            // lastTwo 接过 lastOne 的位置（成为新的 dp[k-1]）
            // lastOne 更新为 curMax（成为新的 dp[k]）
            lastTwo = lastOne;
            lastOne = curMax;
        }

        return lastOne;
    }
};
```

### 4.3 变量滚动过程演示

以 `nums = [2, 7, 9, 3, 1]` 为例：

| 轮次 | num | lastTwo (dp[k-2]) | lastOne (dp[k-1]) | curMax = max(lastOne, lastTwo+num) | 操作 |
|------|-----|-------------------|-------------------|------------------------------------|------|
| 初始 | — | 0 | 0 | — | — |
| 1 | 2 | 0 | **2** | max(0, 0+2) = 2 | lastTwo=0, lastOne=2 |
| 2 | 7 | 2 | **7** | max(2, 0+7) = 7 | lastTwo=2, lastOne=7 |
| 3 | 9 | 7 | **11** | max(7, 2+9) = 11 | lastTwo=7, lastOne=11 |
| 4 | 3 | 11 | **11** | max(11, 7+3) = 11 | lastTwo=11, lastOne=11 |
| 5 | 1 | 11 | **12** | max(11, 11+1) = 12 | lastTwo=11, lastOne=12 |

结果：`lastOne = 12`

### 4.4 为什么赋值顺序不能反？

```cpp
// ✅ 正确顺序：先更新 lastTwo，再更新 lastOne
lastTwo = lastOne;   // lastTwo 先接过旧的 lastOne
lastOne = curMax;    // lastOne 再更新为新值

// ❌ 错误顺序：
lastOne = curMax;    // lastOne 被覆盖，旧值丢失
lastTwo = lastOne;   // lastTwo 得到的是 curMax，不是旧的 lastOne！
```

**记忆口诀**：先"后"再"前"，旧值传给后面，新值赋给前面。

---

## 五、两种解法的本质联系

```
解法一（dp数组）          解法二（滚动变量）
    ↓                         ↓
dp[0] = 0              lastTwo = 0  （初始）
dp[1] = nums[0]        lastOne = 0  （初始）
    ↓                         ↓
dp[2] = max(dp[1],     curMax = max(lastOne,
        dp[0]+nums[1])          lastTwo+nums[0])
    ↓                         ↓
dp[3] = max(dp[2],     curMax = max(lastOne,
        dp[1]+nums[2])          lastTwo+nums[1])
    ↓                         ↓
   ...                       ...
```

**本质相同**：都是 `f(k) = max(f(k-1), f(k-2) + nums[k-1])`，只是存储方式不同。

---

## 六、状态转移方程总结

```
┌─────────────────────────────────────────────┐
│                                             │
│   dp[k] = max(dp[k-1], dp[k-2] + nums[k-1]) │
│                                             │
│   不偷当前房子      偷当前房子               │
│   （继承上一状态）   （上上状态 + 当前金额）   │
│                                             │
└─────────────────────────────────────────────┘
```

**边界条件**：
- `dp[0] = 0`（0 间房子）
- `dp[1] = nums[0]`（1 间房子）

---

## 七、复杂度分析

| 解法 | 时间复杂度 | 空间复杂度 | 说明 |
|------|-----------|-----------|------|
| DP 数组 | O(n) | **O(n)** | 需要存储 n+1 个状态 |
| 滚动变量 | O(n) | **O(1)** ⭐ | 只用两个变量，面试推荐写法 |

---

## 八、易错点 & 踩坑

### ❌ 错误 1：dp 数组和 nums 索引混用
```cpp
// ❌ 错误：dp[i] 对应 nums[i]，但 dp 比 nums 多一个元素
for (int i = 2; i < n; i++) {
    dp[i] = max(dp[i-1], dp[i-2] + nums[i]);  // 索引错位！
}

// ✅ 正确：dp[i] 对应 nums[i-1]
for (int i = 2; i <= n; i++) {
    dp[i] = max(dp[i-1], dp[i-2] + nums[i-1]);
}
```

### ❌ 错误 2：滚动变量赋值顺序反了
```cpp
// ❌ 错误：lastTwo 得到的是新值
lastOne = curMax;
lastTwo = lastOne;  // 此时 lastOne 已经是 curMax 了

// ✅ 正确：先保存旧值
lastTwo = lastOne;
lastOne = curMax;
```

### ❌ 错误 3：忘记处理空数组
```cpp
// ❌ 可能越界
return dp[n];  // 如果 n==0，dp[0] 访问可能出问题

// ✅ 提前判断
if (n == 0) return 0;
```

---

## 九、变形与拓展

### 9.1 力扣 213. 打家劫舍 II（房屋围成环）

首尾房间相邻，不能同时偷。拆成两种情况：
- 偷第一家 → 不能偷最后一家 → 范围 `[0, n-2]`
- 不偷第一家 → 范围 `[1, n-1]`

取两种情况的最大值。

### 9.2 力扣 337. 打家劫舍 III（二叉树形）

房子排成二叉树，父子节点不能同时偷。用**树形 DP**：
```cpp
// 每个节点返回两个值：
// (偷当前节点的最大金额, 不偷当前节点的最大金额)
pair<int, int> dfs(TreeNode* node) {
    if (!node) return {0, 0};
    auto left = dfs(node->left);
    auto right = dfs(node->right);

    int rob = node->val + left.second + right.second;      // 偷当前
    int notRob = max(left.first, left.second) + 
                 max(right.first, right.second);           // 不偷当前

    return {rob, notRob};
}
```

---

## 十、总结

| 要点 | 内容 |
|------|------|
| **状态定义** | `dp[k]` = 前 k 个房间能偷到的最大金额 |
| **转移方程** | `dp[k] = max(dp[k-1], dp[k-2] + nums[k-1])` |
| **边界条件** | `dp[0] = 0`, `dp[1] = nums[0]` |
| **空间优化** | 用两个变量代替 dp 数组，O(1) 空间 |
| **面试推荐** | 先写 DP 数组版本确保正确，再优化为滚动变量 |

> 💡 **一句话总结**：每间房子选不选？不选就继承上一个最优解，选就加上隔一个的最优解——取两者最大值。
