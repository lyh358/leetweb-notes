# LeetCode 1. 两数之和 (Two Sum) — 学习笔记

> **难度**: Easy | **标签**: 数组、哈希表 | **推荐指数**: ⭐⭐⭐⭐⭐

---

## 一、题目描述

### 原题

给定一个整数数组 `nums` 和一个整数目标值 `target`，请你在该数组中找出 **和为目标值** 的那 **两个** 整数，并返回它们的数组下标。

你可以假设每种输入只会对应一个答案。但是，数组中同一个元素在答案里不能重复出现。

你可以按任意顺序返回答案。

### 示例

**示例 1：**
```
输入: nums = [2, 7, 11, 15], target = 9
输出: [0, 1]
解释: 因为 nums[0] + nums[1] == 9，所以返回 [0, 1]。
```

**示例 2：**
```
输入: nums = [3, 2, 4], target = 6
输出: [1, 2]
```

**示例 3：**
```
输入: nums = [3, 3], target = 6
输出: [0, 1]
```

### 约束条件
- `2 <= nums.length <= 10^4`
- `-10^9 <= nums[i] <= 10^9`
- `-10^9 <= target <= 10^9`
- 只会存在一个有效答案

---

## 二、题目分析

### 核心问题
在数组中找到两个数，使它们的和等于 `target`，并返回这两个数的**下标**。

### 关键点
1. **返回的是下标，不是数值** — 排序后下标会改变，需要额外处理
2. **同一个元素不能使用两次** — 例如 `[3, 3]` 中两个 3 的下标分别是 0 和 1，这是允许的，但不能用同一个下标两次
3. **假设有且仅有一个答案** — 不需要处理无解或多解的情况
4. **答案顺序不限** — 返回 `[0, 1]` 或 `[1, 0]` 都算正确

---

## 三、思路梳理

### 思路一：暴力枚举（Brute Force）

**想法**：直接枚举所有可能的两个数的组合，检查它们的和是否等于 `target`。

- 外层循环遍历第一个数 `nums[i]`
- 内层循环遍历第二个数 `nums[j]`（`j > i`，避免重复）
- 如果 `nums[i] + nums[j] == target`，返回 `[i, j]`

### 思路二：哈希表（Hash Map）— 两遍扫描

**想法**：先建立值到下标的映射，再查找。

- 第一遍：遍历数组，将每个值 `nums[i]` 作为 key，下标 `i` 作为 value，存入哈希表
- 第二遍：再次遍历数组，对于每个 `nums[i]`，计算 `complement = target - nums[i]`
  - 如果 `complement` 在哈希表中，且其下标不等于 `i`，则找到答案

### 思路三：哈希表（Hash Map）— 一遍扫描 ⭐ 最优解

**想法**：在遍历的同时查找，边存边查。

- 遍历数组，对于当前元素 `nums[i]`：
  - 计算 `complement = target - nums[i]`
  - 如果 `complement` 已经在哈希表中，说明之前存过的某个数和当前数之和为 `target`，直接返回答案
  - 否则，将 `nums[i]` 及其下标 `i` 存入哈希表，继续遍历

---

## 四、解题过程（C++）

### 解法一：暴力枚举

```cpp
class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        int n = nums.size();
        for (int i = 0; i < n; i++) {
            for (int j = i + 1; j < n; j++) {
                if (nums[i] + nums[j] == target) {
                    return {i, j};
                }
            }
        }
        return {};
    }
};
```

---

### 解法二：哈希表 — 两遍扫描

```cpp
class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        unordered_map<int, int> hash_map;

        // 第一遍：建立哈希表
        for (int i = 0; i < nums.size(); i++) {
            hash_map[nums[i]] = i;
        }

        // 第二遍：查找 complement
        for (int i = 0; i < nums.size(); i++) {
            int complement = target - nums[i];
            if (hash_map.count(complement) && hash_map[complement] != i) {
                return {i, hash_map[complement]};
            }
        }

        return {};
    }
};
```

---

### 解法三：哈希表 — 一遍扫描 ⭐ 推荐

```cpp
class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        unordered_map<int, int> hash_map;  // 值 -> 下标

        for (int i = 0; i < nums.size(); i++) {
            int complement = target - nums[i];
            if (hash_map.count(complement)) {
                // 找到 complement，且它是在当前元素之前出现的
                return {hash_map[complement], i};
            }
            // 将当前元素存入哈希表，供后面的元素查找
            hash_map[nums[i]] = i;
        }

        return {};
    }
};
```

---

## 五、涉及的方法与知识点

### 1. 哈希表（Hash Map / Hash Table）

- **定义**：一种通过键（key）直接访问值（value）的数据结构
- **核心操作**：
  - `插入`：O(1) 平均时间复杂度
  - `查找`：O(1) 平均时间复杂度
  - `删除`：O(1) 平均时间复杂度
- **在本题中的应用**：将已经遍历过的数值及其下标存入哈希表，实现 O(1) 的查找
- **C++ 中使用 `unordered_map`**：基于哈希表实现，平均 O(1) 操作；`count(key)` 用于判断 key 是否存在

### 2. 枚举思想

- **暴力枚举**：不借助额外数据结构，直接尝试所有可能的组合
- **有序枚举**：通过控制循环变量（如 `j > i`）避免重复计算

### 3. 空间换时间

- 暴力解法时间复杂度高（O(n²)），空间复杂度低（O(1)）
- 哈希表解法通过额外 O(n) 的空间，将时间复杂度降到 O(n)
- 这是算法优化中非常经典的 **Trade-off（权衡）**

### 4. 补数思想（Complement）

- 对于当前数 `num`，要找的另一个数就是 `target - num`
- 将"两数之和"问题转化为"查找补数是否存在的问题"

---

## 六、复杂度分析

| 解法 | 时间复杂度 | 空间复杂度 | 说明 |
|------|-----------|-----------|------|
| 暴力枚举 | O(n²) | O(1) | 双重循环，无额外空间 |
| 哈希表两遍扫描 | O(n) | O(n) | 两次遍历，建立哈希表 |
| 哈希表一遍扫描 | O(n) | O(n) | ⭐ 最优解，一次遍历 |

### 详细分析

**暴力枚举：**
- 外层循环执行 n 次，内层循环平均执行 n/2 次
- 总比较次数约为 n × (n-1) / 2，即 O(n²)
- 只使用了常数个变量，空间复杂度 O(1)

**哈希表解法：**
- 遍历数组一次（或两次），每次哈希表操作平均 O(1)
- 时间复杂度为 O(n)
- 哈希表最多存储 n 个元素，空间复杂度为 O(n)
- 最坏情况下（哈希冲突严重）时间复杂度可能退化为 O(n²)，但实际中极少发生

---

## 七、易错点与注意事项

### ⚠️ 1. 返回的是下标，不是数值
```cpp
// ❌ 错误：返回了数值
return {nums[i], nums[j]};

// ✅ 正确：返回下标
return {i, j};
```

### ⚠️ 2. 同一个元素不能使用两次
```cpp
// 两遍扫描时，需要检查下标是否相同
if (hash_map.count(complement) && hash_map[complement] != i)
```

### ⚠️ 3. 数组中有重复元素
例如 `nums = [3, 3], target = 6`，哈希表中后面的 3 会覆盖前面的 3。
- **一遍扫描**：不会出问题，因为找到 complement 时，第一个 3 已经作为 complement 被找到了
- **两遍扫描**：如果直接覆盖，可能会丢失信息。但在本题中，由于答案唯一且两个 3 的下标不同，通常不影响。更安全的做法是用 `hash_map[num] = i` 覆盖即可，因为题目保证有唯一解

### ⚠️ 4. 负数处理
题目中数值可能为负数，但哈希表可以正常处理，无需特殊逻辑。

### ⚠️ 5. C++ 中 `unordered_map` vs `map`
- `unordered_map`：基于哈希表，平均 O(1)，**推荐用于本题**
- `map`：基于红黑树，O(log n)，有序但不适合本题场景

---

## 八、举一反三

### 类似题目

| 题目 | 难度 | 核心变化 |
|------|------|---------|
| [15. 三数之和](https://leetcode.cn/problems/3sum/) | Medium | 三个数之和为 0，需去重，用排序+双指针 |
| [16. 最接近的三数之和](https://leetcode.cn/problems/3sum-closest/) | Medium | 找最接近 target 的三数之和 |
| [18. 四数之和](https://leetcode.cn/problems/4sum/) | Medium | 四个数之和为 target |
| [167. 两数之和 II - 输入有序数组](https://leetcode.cn/problems/two-sum-ii-input-array-is-sorted/) | Medium | 数组已排序，可用双指针，空间 O(1) |
| [170. 两数之和 III - 数据结构设计](https://leetcode.cn/problems/two-sum-iii-data-structure-design/) | Easy | 设计类，支持 add 和 find 操作 |
| [653. 两数之和 IV - 输入 BST](https://leetcode.cn/problems/two-sum-iv-input-is-a-bst/) | Easy | 在二叉搜索树中找两数之和 |

### 扩展思路

1. **如果数组已排序**：可以使用 **双指针** 法，空间复杂度可优化到 O(1)
2. **如果需要找出所有组合**：需要处理去重问题，通常先排序
3. **如果数据量极大，内存有限**：可以考虑 **排序 + 双指针**，空间更优

---

## 九、总结

| 要点 | 内容 |
|------|------|
| **核心技巧** | 哈希表 + 补数思想 |
| **最优解法** | 一遍扫描 `unordered_map`，时间 O(n)，空间 O(n) |
| **关键洞察** | 将「找两个数」转化为「找补数是否存在」 |
| **Trade-off** | 用 O(n) 空间换取 O(n) 时间，避免 O(n²) 暴力 |
| **适用场景** | 数组无序、需要返回原始下标 |

> 💡 **学习心得**：两数之和是哈希表应用的经典入门题。掌握"空间换时间"和"补数思想"后，可以顺利过渡到三数之和、四数之和等更复杂的题目。在 C++ 中优先使用 `unordered_map` 实现 O(1) 的查找效率。

---

*笔记整理时间: 2026-07-29*
