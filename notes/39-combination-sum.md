# 🌲 回溯算法学习笔记 —— 力扣 39. 组合总和（选/不选版）

> 基于「选 / 不选」二叉决策树模型的 DFS 回溯解法

---

## 📌 一、题目：LeetCode 39. 组合总和

**题目链接**：[39. 组合总和](https://leetcode.cn/problems/combination-sum/)

**题目描述**：给你一个**无重复元素**的整数数组 `candidates` 和一个目标整数 `target`，找出 `candidates` 中可以使数字和为目标数 `target` 的所有**不同**组合。同一个数字可以被**无限制重复选取**。

**示例**：

```
输入: candidates = [2,3,6,7], target = 7
输出: [[2,2,3],[7]]
解释:
2 + 2 + 3 = 7，注意 2 可以使用多次
7 = 7
```

**约束**：
- `1 <= candidates.length <= 30`
- `2 <= candidates[i] <= 40`
- `candidates` 的所有元素**互不相同**
- `1 <= target <= 40`

---

## 🧠 二、核心思路：「选 / 不选」二叉决策树

你的代码采用了与子集问题（LeetCode 78）相同的**二叉决策树**模型：

> 对于数组中的**每一个元素**，我们只有两种选择 —— **选它** 或者 **不选它**。

但与子集问题不同的是，这道题有两个额外的关键约束：

1. **可以重复选** → 选了当前数之后，下一层**仍然可以选它**
2. **目标和** → 需要一个 `target` 变量来跟踪剩余需要凑的数值

### 递归参数设计

```cpp
void backtracing(ans, combination, candidates, target, idx)
```

| 参数 | 含义 |
|-----|------|
| `ans` | 存储所有合法组合的二维结果数组 |
| `combination` | 当前正在构造的一维组合 |
| `candidates` | 候选数字数组 |
| `target` | **剩余还需要凑的目标值**（关键状态变量） |
| `idx` | 当前正在考虑是否选取的元素下标 |

### 为什么用「选/不选」而不是 for 循环？

两种写法都能解决问题，但思维角度不同：

| 写法 | 思维模型 | 递归树形状 | 适用场景 |
|-----|---------|-----------|---------|
| **选/不选（你的写法）** | 二叉决策树 | 每个节点分两支 | 子集、组合总和（逻辑直观） |
| **for 循环** | 多叉决策树 | 每个节点分多支 | 组合、排列（通用模板） |

> 「选/不选」版的优势：**逻辑极其清晰**——每个元素只有两种命运，像开关一样。

---

## 💻 三、完整代码（你的代码，逐行解析）

```cpp
class Solution {
public:
    // 一级二维结果数组和二级一维结果数组
    vector<vector<int>> ans;
    vector<int> combination;

    // 回溯函数
    void backtracing(vector<vector<int>>& ans,
                     vector<int>& combination,
                     vector<int>& candidates,
                     int target,
                     int idx)
    {
        // ========== Step 1: 退出条件设定 ==========

        // 退出条件 1：指针越界，所有数都考虑完了
        // 还没找到和为 target 的组合，直接返回不记录
        if (idx == candidates.size()) return;

        // 退出条件 2：找到了和为 target 的组合
        // target 被减到 0，说明当前 combination 恰好满足条件
        if (target == 0) 
        {
            ans.push_back(combination);
            return;
        }

        // ========== Step 2: 回溯核心操作 ==========

        // -------- 分支 1：不选当前数 --------
        // 不选 candidates[idx]，target 不变，考虑下一个数
        backtracing(ans, combination, candidates, target, idx + 1);

        // -------- 分支 2：选当前数 --------
        // 只有在选了之后总和不超过 target 的情况下才能选
        if (target - candidates[idx] >= 0)
        {
            // 选了就先加入组合
            combination.push_back(candidates[idx]);

            // 因为可以重复选，所以：
            // - target 变小（减去当前选的数）
            // - idx 不变（下一轮还可以继续选当前这个数！）
            backtracing(ans, combination, candidates, 
                       target - candidates[idx], idx);

            // 回溯：撤销选择
            combination.pop_back();
        }
    }

    vector<vector<int>> combinationSum(vector<int>& candidates, int target) {
        backtracing(ans, combination, candidates, target, 0);
        return ans;
    }
};
```

---

## 🌳 四、递归树可视化（以 candidates=[2,3,6,7], target=7 为例）

```
                              target=7, idx=0 (考虑 2)
                             /                        \
                       不选 2                          选 2
                      /                                \
              target=7, idx=1                    target=5, idx=0 (还能选 2！)
             /          \                         /                \
          不选3        选3                  不选2              选2
         /            \                  /                  \
    target=7,idx=2  target=4,idx=2  target=5,idx=1      target=3,idx=0
    (考虑6)         (考虑6)         (考虑3)             (还能选2！)
       ...            ...            ...                  ...
```

**关键路径演示**：

| 步骤 | 操作 | combination | target | idx | 说明 |
|-----|------|------------|--------|-----|------|
| 1 | 选 2 | `[2]` | 5 | 0 | target=7-2=5，idx不变（还能选2） |
| 2 | 选 2 | `[2,2]` | 3 | 0 | target=5-2=3，idx不变（还能选2） |
| 3 | 选 2 | `[2,2,2]` | 1 | 0 | target=3-2=1，idx不变 |
| 4 | 选 2？ | — | — | 0 | target-candidates[0]=1-2=-1 < 0，**不能选** |
| 5 | 不选 2 | `[2,2,2]` | 1 | 1 | idx+1，考虑3 |
| 6 | 选 3？ | — | — | 1 | target-candidates[1]=1-3=-2 < 0，**不能选** |
| 7 | 不选 3 | `[2,2,2]` | 1 | 2 | idx+1，考虑6 |
| ... | ... | ... | ... | ... | 一路不选到底，idx越界返回 ❌ |
| 8 | **回溯**到 `[2,2]` | `[2,2]` | 3 | 0 | pop掉第三个2 |
| 9 | 不选 2 | `[2,2]` | 3 | 1 | 考虑3 |
| 10 | 选 3 | `[2,2,3]` | 0 | 1 | target=3-3=0 ✅ |
| 11 | target==0 | — | — | — | **记录 [2,2,3]** ✅ |

---

## ❓ 五、核心问题解析

### 5.1 为什么「选」的时候 idx 不变，「不选」的时候 idx+1？

这是这道题**最精妙**的地方：

```cpp
// 不选：idx + 1 → 考虑下一个数，当前数不再使用
backtracing(ans, combination, candidates, target, idx + 1);

// 选：idx 不变 → 下一轮还能继续选当前这个数（重复选取！）
backtracing(ans, combination, candidates, target - candidates[idx], idx);
```

| 分支 | idx 变化 | 含义 |
|-----|---------|------|
| **不选** | `idx + 1` | 当前数被跳过，以后也不再考虑它 |
| **选** | `idx`（不变） | 当前数被加入组合，**下一轮还可以再选它** |

> 💡 **一句话**：`idx` 不变 = 「我还没用完这个数，还能继续拿」；`idx+1` = 「这个数我不要了，看下一个」。

### 5.2 为什么先「不选」再「选」？

你的代码中先递归「不选」分支，再处理「选」分支。这个顺序**不影响最终结果**，但会影响结果在 `ans` 中的顺序：

```cpp
// 先不选
backtracing(..., idx + 1);           // 左分支
// 后选
if (...) { backtracing(..., idx); }  // 右分支
```

**如果反过来**（先选后不选），逻辑上也可以，但通常「选/不选」模型中：
- 先「选」后「不选」：先生成包含当前元素的组合
- 先「不选」后「选」：先生成不包含当前元素的组合

两种顺序都正确，只是遍历顺序不同。

### 5.3 target == 0 和 idx 越界，哪个终止条件先判断？

你的代码中：

```cpp
if (idx == candidates.size()) return;   // 条件1
if (target == 0) { ans.push_back(...); } // 条件2
```

**顺序很重要！** 必须先判断 `idx` 越界，再判断 `target == 0`。

**为什么？**
- 如果先判断 `target == 0`，当 `target` 恰好为 0 且 `idx` 也恰好越界时，会正确记录结果。
- 但如果 `target` 不为 0 且 `idx` 越界了，必须先返回，否则会继续执行下面的「选/不选」逻辑，导致数组越界访问。

实际上，更安全的写法是：

```cpp
if (target == 0) { ans.push_back(combination); return; }  // 先判断成功
if (idx == candidates.size()) return;                      // 再判断失败
```

这样当 `target==0` 时直接收集结果并返回，不需要再判断 `idx`。

### 5.4 为什么不需要排序也能工作？

你的代码**没有排序**，也能正确运行。这是因为：

- 「选/不选」模型天然会枚举所有可能的组合
- `if (target - candidates[idx] >= 0)` 这个条件已经起到了一定的剪枝作用

**但排序后效率更高**：
- 排序后可以在 `idx` 越界判断之前增加 `if (target < 0) return;` 的剪枝
- 或者像 for 循环版那样在循环中 `break`

---

## ⚔️ 六、「选/不选」版 vs 「for循环」版对比

| 对比维度 | **选/不选版（你的代码）** | **for 循环版** |
|---------|------------------------|---------------|
| **思维模型** | 二叉决策树（每个元素二选一） | 多叉决策树（从候选集中枚举） |
| **递归参数** | `idx`（当前考虑哪个元素） | `start`（从哪个位置开始枚举） |
| **重复选取实现** | 「选」分支 `idx` 不变 | `backtrack(i)`（传 `i` 而非 `i+1`） |
| **代码行数** | 较少，逻辑清晰 | 较多，但更通用 |
| **剪枝难度** | 稍难（需在递归前判断） | 较易（排序后循环内 `break`） |
| **适用场景** | 子集、组合总和（逻辑直观） | 组合、排列、复杂约束（通用模板） |
| **结果顺序** | 按「不选优先」深度遍历 | 按「从小到大」组合顺序 |

### 两种写法的代码对比

**选/不选版（你的代码）**：
```cpp
void backtrack(target, idx) {
    if (idx == n) return;
    if (target == 0) { ans.push_back(path); return; }

    // 不选
    backtrack(target, idx + 1);

    // 选（如果可以）
    if (target - candidates[idx] >= 0) {
        path.push_back(candidates[idx]);
        backtrack(target - candidates[idx], idx);  // idx 不变！
        path.pop_back();
    }
}
```

**for 循环版**：
```cpp
void backtrack(target, start) {
    if (target == 0) { ans.push_back(path); return; }
    if (target < 0) return;  // 剪枝

    for (int i = start; i < n; i++) {
        path.push_back(candidates[i]);
        backtrack(target - candidates[i], i);  // 传 i，不是 i+1！
        path.pop_back();
    }
}
```

---

## ⚠️ 七、常见易错点

| 坑点 | 错误写法 | 正确写法 | 后果 |
|-----|---------|---------|------|
| **选的时候 idx+1** | `backtrack(target - x, idx + 1)` | `backtrack(target - x, idx)` | 无法重复选取同一个数，漏解 |
| **不选的时候 idx 不变** | `backtrack(target, idx)` | `backtrack(target, idx + 1)` | 无限递归，死循环 |
| **target 没更新** | 两个分支都传 `target` | 「选」分支传 `target - candidates[idx]` | target 永远是初始值，无法终止 |
| **终止条件顺序错误** | 先判断 `target==0` | 先判断 `idx` 越界 | 可能导致数组越界访问 |
| **忘记回溯 pop** | 只 push 不 pop | push 后必须 pop | combination 状态混乱，结果错误 |
| **选之前没判断** | 直接 push + 递归 | 先判断 `target - candidates[idx] >= 0` | 可能产生负数和的组合 |

---

## 📊 八、复杂度分析

- **时间复杂度**：`O(S)`，其中 `S` 为所有可行解的长度之和
  - 最坏情况下需要枚举大量组合
  - 「选/不选」模型会遍历整棵二叉决策树

- **空间复杂度**：`O(target / min(candidates))`
  - 递归栈深度取决于最小元素能被选多少次
  - 例如 `target=7, min=2`，最深递归约 3-4 层

---

## 🚀 九、举一反三：组合总和家族

| 题目 | 难度 | 与 39 的区别 | 能否用「选/不选」模型 |
|-----|------|------------|-------------------|
| **39. 组合总和** | 中等 | 基础版，无重复元素，可重复选 | ✅ 可以（你的写法） |
| **40. 组合总和 II** | 中等 | 数组含**重复元素**，每个数只能选一次 | ⚠️ 较复杂，需要同层去重 |
| **216. 组合总和 III** | 中等 | 范围固定 `[1,9]`，选 k 个数和为 n | ✅ 可以 |
| **377. 组合总和 Ⅳ** | 中等 | 求**排列数**（顺序不同算不同） | ❌ 用动态规划 |
| **78. 子集** | 中等 | 所有子集，无 target 约束 | ✅ 非常适合 |
| **77. 组合** | 中等 | 选 k 个，不可重复 | ⚠️ 需要额外维护 size |

---

## 💡 十、学习建议

1. **理解 `idx` 的双重作用**
   - 「不选」时 `idx+1`：当前数用完，看下一个
   - 「选」时 `idx` 不变：当前数还能继续用
   - 这是实现「可重复选取」的关键技巧

2. **画二叉决策树**
   - 在纸上画出 `candidates=[2,3], target=5` 的完整递归树
   - 标注每个节点的 `target` 和 `idx` 值
   - 理解 `[2,3]` 和 `[3,2]` 为什么只会生成一个

3. **对比两种模型**
   - 「选/不选」版：逻辑清晰，适合理解回溯本质
   - for 循环版：更通用，适合面试快速写出
   - 建议两种都会，根据题目选择

4. **注意 target 的传递**
   - 「选/不选」模型中，target 是**剩余目标值**
   - 每次「选」都要更新：`target - candidates[idx]`
   - 这是与纯组合问题最大的区别

---

## 🎯 一句话总结

> **组合总和的「选/不选」模型 = 子集的二叉决策树 + target 递减约束**
>
> 「不选」→ `idx+1`，target 不变，跳过当前数；  
> 「选」→ `idx` 不变，target 减小，**同一个数还能再选**；  
> `target == 0` 时收集结果，`idx` 越界时直接返回。

---

## 📎 附录：「选/不选」模型速查卡

```cpp
// 通用模板：选/不选模型
void backtrack(参数, idx) {
    // 终止条件（失败）
    if (越界/不满足) return;

    // 终止条件（成功）
    if (满足条件) { 收集结果; return; }

    // 分支 1：不选当前元素
    backtrack(参数不变, idx + 1);

    // 分支 2：选当前元素（如果可以）
    if (满足选取条件) {
        path.push_back(x);
        backtrack(参数更新, idx);      // idx 不变 = 可重复选
        // 或 backtrack(参数更新, idx + 1); // idx+1 = 不可重复选
        path.pop_back();
    }
}
```

| 问题 | 不选分支 | 选分支 | 终止条件 |
|-----|---------|--------|---------|
| **子集（78）** | `idx+1` | `idx+1` | `idx == n`（收集） |
| **组合（77）** | `idx+1` | `idx+1` | `size == k`（收集） |
| **组合总和（39）** | `idx+1, target不变` | `idx, target-x` | `target==0`（收集） |

---

> 祝你刷题顺利，早日掌握回溯的精髓！🎉
