# LeetCode 34. 在排序数组中查找元素的第一个和最后一个位置 —— 学习笔记

> **题目**: [34. Find First and Last Position of Element in Sorted Array](https://leetcode.cn/problems/find-first-and-last-position-of-element-in-sorted-array/)  
> **核心**: 给定一个按照**非降序排列**的整数数组 `nums`，和一个目标值 `target`。找出 `target` 在数组中的**开始位置**和**结束位置**。  
> 如果数组中不存在 `target`，返回 `[-1, -1]`。  
> **要求**：必须设计时间复杂度为 $O(\log n)$ 的算法。  
> **示例**:  
> ```
> nums = [5, 7, 7, 8, 8, 10], target = 8  →  [3, 4]
> nums = [5, 7, 7, 8, 8, 10], target = 6  →  [-1, -1]
> nums = [], target = 0                     →  [-1, -1]
> ```

---

## 一、核心思想：两次特化二分查找

### 1.1 为什么普通二分查找不够用？
普通二分查找在 `nums[mid] == target` 时直接 `return mid`，它返回的是**任意一个**等于 target 的位置。但本题要求找到**最左边**和**最右边**的两个边界。

如果数组中有多个重复的 target，普通二分"碰到就返回"的策略无法保证返回的是边界位置。

### 1.2 解题策略
将问题拆成两个独立的子问题：
1. **找最左边的 target**（lower_bound）
2. **找最右边的 target**（upper_bound 的前一个）

两次都使用二分查找，但**命中时的处理不同**——不立即返回，而是继续向边界方向收缩。

---

## 二、代码实现（带完整注释）

```cpp
class Solution {
public:
    vector<int> searchRange(vector<int>& nums, int target) {
        int n = nums.size();

        // ========== 初始化结果 ==========
        // first: 最左边 target 的位置，找不到保持 -1
        // last:  最右边 target 的位置，找不到保持 -1
        int first = -1, last = -1;

        // ========== 第一次二分：找最左边的 target ==========
        int left = 0, right = n - 1;
        while (left <= right) {
            int mid = left + (right - left) / 2;

            if (nums[mid] == target) {
                // 找到了一个等于 target 的位置
                // 但不确定是不是最左边，先记录下来
                first = mid;
                // 关键：继续向左搜索，看看左边还有没有 target
                // 将右边界移到 mid - 1，逼使搜索范围向左收缩
                right = mid - 1;
            }
            else if (nums[mid] < target) {
                // mid 及左侧都太小，去右半部分找
                left = mid + 1;
            }
            else {
                // mid 及右侧都太大，去左半部分找
                right = mid - 1;
            }
        }

        // ========== 第二次二分：找最右边的 target ==========
        left = 0, right = n - 1;
        while (left <= right) {
            int mid = left + (right - left) / 2;

            if (nums[mid] == target) {
                // 找到了一个等于 target 的位置
                // 但不确定是不是最右边，先记录下来
                last = mid;
                // 关键：继续向右搜索，看看右边还有没有 target
                // 将左边界移到 mid + 1，逼使搜索范围向右收缩
                left = mid + 1;
            }
            else if (nums[mid] < target) {
                left = mid + 1;
            }
            else {
                right = mid - 1;
            }
        }

        return {first, last};
    }
};
```

---

## 三、关键机制深度解析

### 3.1 找最左边：为什么命中时 `right = mid - 1`？

当 `nums[mid] == target` 时，说明 `mid` 是一个候选答案，但它**不一定是最左边**的。为了找到更左边的 target，我们需要**继续在左半部分搜索**。

```
数组: [5, 7, 7, 7, 8, 8, 10], target = 7

假设 mid 指向索引 3（值为 7）:
    [5, 7, 7, 7, 8, 8, 10]
             ↑
            mid=3

普通二分: 直接返回 3 ❌（不是最左边）
本题策略: first = 3 暂存，然后 right = 2，继续在 [0, 2] 中搜索

下一轮 mid 指向索引 1（值为 7）:
    [5, 7, 7]
        ↑
       mid=1
    first = 1, right = 0，继续在 [0, 0] 中搜索

再下一轮 mid 指向索引 0（值为 5）:
    [5]
     ↑
    mid=0, nums[0]=5 < 7, left = 1

此时 left=1, right=0，循环结束。first = 1 即为最左边 ✓
```

**核心逻辑**：即使命中了，也不停止，而是把右边界压到 `mid - 1`，逼使搜索向左走。如果左边还有 target，一定会被找到并更新 `first`；如果没有，循环自然结束，`first` 保持为已找到的最左位置。

### 3.2 找最右边：为什么命中时 `left = mid + 1`？

同理，当 `nums[mid] == target` 时，`mid` 是候选答案，但不一定是最右边。为了找到更右边的 target，需要**继续在右半部分搜索**。

```
数组: [5, 7, 7, 7, 8, 8, 10], target = 7

假设 mid 指向索引 3（值为 7）:
    last = 3 暂存，left = 4，继续在 [4, 6] 中搜索

下一轮 mid 指向索引 5（值为 8）:
    nums[5]=8 > 7, right = 4

再下一轮 mid 指向索引 4（值为 8）:
    nums[4]=8 > 7, right = 3

此时 left=4, right=3，循环结束。last = 3 即为最右边 ✓
```

### 3.3 两次查找的独立性

两次二分查找**完全独立**：
- 第一次只负责找 `first`，不管 `last`；
- 第二次只负责找 `last`，不管 `first`。

即使数组中没有 target：
- 第一次：`first` 始终未被赋值，保持 `-1`；
- 第二次：`last` 始终未被赋值，保持 `-1`。

最终返回 `[-1, -1]`，符合题意。

---

## 四、逐步推演示例

### 示例 1：target 存在且有多个重复
```
nums = [5, 7, 7, 8, 8, 10], target = 8
```

**找 first（最左边）**：

| 轮次 | left | right | mid | nums[mid] | 操作 | first |
|:----:|:----:|:-----:|:---:|:---------:|:----:|:-----:|
| 初始 | 0 | 5 | — | — | — | -1 |
| 1 | 0 | 5 | 2 | nums[2]=7 | 7 < 8, left=3 | -1 |
| 2 | 3 | 5 | 4 | nums[4]=8 | **命中!** first=4, right=3 | 4 |
| 3 | 3 | 3 | 3 | nums[3]=8 | **命中!** first=3, right=2 | 3 |
| — | 3 | 2 | — | left > right，结束 | — | **3** |

**找 last（最右边）**：

| 轮次 | left | right | mid | nums[mid] | 操作 | last |
|:----:|:----:|:-----:|:---:|:---------:|:----:|:----:|
| 初始 | 0 | 5 | — | — | — | -1 |
| 1 | 0 | 5 | 2 | nums[2]=7 | 7 < 8, left=3 | -1 |
| 2 | 3 | 5 | 4 | nums[4]=8 | **命中!** last=4, left=5 | 4 |
| 3 | 5 | 5 | 5 | nums[5]=10 | 10 > 8, right=4 | 4 |
| — | 5 | 4 | — | left > right，结束 | — | **4** |

**结果**: `[3, 4]` ✓

---

### 示例 2：target 不存在
```
nums = [5, 7, 7, 8, 8, 10], target = 6
```

**找 first**：

| 轮次 | left | right | mid | nums[mid] | 操作 | first |
|:----:|:----:|:-----:|:---:|:---------:|:----:|:-----:|
| 1 | 0 | 5 | 2 | 7 | 7 > 6, right=1 | -1 |
| 2 | 0 | 1 | 0 | 5 | 5 < 6, left=1 | -1 |
| 3 | 1 | 1 | 1 | 7 | 7 > 6, right=0 | -1 |
| — | 1 | 0 | — | 结束 | — | **-1** |

**找 last**：同理，`last` 也保持 `-1`。

**结果**: `[-1, -1]` ✓

---

### 示例 3：target 只有一个
```
nums = [1, 2, 3, 4, 5], target = 3
```

**找 first**：

| 轮次 | left | right | mid | nums[mid] | 操作 | first |
|:----:|:----:|:-----:|:---:|:---------:|:----:|:-----:|
| 1 | 0 | 4 | 2 | 3 | **命中!** first=2, right=1 | 2 |
| 2 | 0 | 1 | 0 | 1 | 1 < 3, left=1 | 2 |
| 3 | 1 | 1 | 1 | 2 | 2 < 3, left=2 | 2 |
| — | 2 | 1 | — | 结束 | — | **2** |

**找 last**：

| 轮次 | left | right | mid | nums[mid] | 操作 | last |
|:----:|:----:|:-----:|:---:|:---------:|:----:|:----:|
| 1 | 0 | 4 | 2 | 3 | **命中!** last=2, left=3 | 2 |
| 2 | 3 | 4 | 3 | 4 | 4 > 3, right=2 | 2 |
| — | 3 | 2 | — | 结束 | — | **2** |

**结果**: `[2, 2]` ✓

---

### 示例 4：空数组
```
nums = [], target = 0
```

`n = 0`，`left = 0, right = -1`，循环条件 `0 <= -1` 不成立，直接跳过。

**结果**: `[-1, -1]` ✓

---

## 五、与 C++ STL 的关系

C++ 标准库提供了现成的 lower_bound 和 upper_bound，可以直接使用：

```cpp
class Solution {
public:
    vector<int> searchRange(vector<int>& nums, int target) {
        // lower_bound: 返回第一个 >= target 的迭代器
        auto left = lower_bound(nums.begin(), nums.end(), target);

        // upper_bound: 返回第一个 > target 的迭代器
        auto right = upper_bound(nums.begin(), nums.end(), target);

        // 检查是否找到
        if (left == right) {
            // lower_bound == upper_bound 说明没有 target
            return {-1, -1};
        }

        // 转换为下标返回
        return {(int)(left - nums.begin()), (int)(right - nums.begin() - 1)};
    }
};
```

| 函数 | 返回值 | 本题对应 |
|:----:|:------:|:--------:|
| `lower_bound` | 第一个 `>= target` 的位置 | `first`（当 target 存在时） |
| `upper_bound` | 第一个 `> target` 的位置 | `last + 1` |

**注意**：`upper_bound - 1` 才是最右边 target 的位置。

> 面试时建议**手写二分**，展示对原理的理解；工程中使用 STL 更简洁。

---

## 六、复杂度分析

| 维度 | 结果 |
|------|------|
| **时间复杂度** | $O(\log n)$ —— 两次二分查找，每次 $O(\log n)$ |
| **空间复杂度** | $O(1)$ —— 只用了常数个变量 |

---

## 七、易错点与面试 Tips

### ❌ 错误 1：找最左边时命中后直接返回
```cpp
if (nums[mid] == target) {
    return mid;  // ❌ 错误！这不是最左边
}
```

### ❌ 错误 2：找最左边时 `left = mid + 1`
```cpp
if (nums[mid] == target) {
    first = mid;
    left = mid + 1;  // ❌ 错误！这会向右找，变成找最右边了
}
```

### ❌ 错误 3：找最右边时 `right = mid - 1`
```cpp
if (nums[mid] == target) {
    last = mid;
    right = mid - 1;  // ❌ 错误！这会向左找，变成找最左边了
}
```

### ❌ 错误 4：忘记初始化 `first` 和 `last` 为 -1
```cpp
int first, last;  // ❌ 未初始化，可能是随机值
```

### ❌ 错误 5：两次查找共用边界变量未重置
```cpp
// 错误：第一次查找后 left/right 已经变了，第二次要重新赋值
while (left <= right) { /* 找 first */ }
// left = 0; right = n - 1;  // ❌ 忘记重置！
while (left <= right) { /* 找 last */ }
```

### ✅ 面试回答框架
1. **思路**：两次二分查找，分别找左边界和右边界；
2. **找左边界**：命中时不返回，记录位置后 `right = mid - 1` 继续向左收缩；
3. **找右边界**：命中时不返回，记录位置后 `left = mid + 1` 继续向右收缩；
4. **复杂度**：时间 $O(\log n)$，空间 $O(1)$；
5. **扩展**：可以提到 STL 的 lower_bound/upper_bound。

### ✅ 追问准备
- **问**：能不能只进行一次二分查找？  
  **答**：可以。先找 lower_bound，然后从该位置线性扫描找右边界。但最坏情况下（全是 target）会退化到 $O(n)$，不满足题目 $O(\log n)$ 要求。

- **问**：如果数组中有无数个 target（虚拟数组），怎么找？  
  **答**：这就是 lower_bound 和 upper_bound 的经典应用场景。

---

## 八、一句话总结

> **普通二分"见好就收"，碰到 target 就返回；找边界要"贪得无厌"，命中了还要往边上蹭——找最左就往左挤（`right = mid - 1`），找最右就往右挤（`left = mid + 1`），直到把边界逼到墙角。**

| 要点 | 找最左边 | 找最右边 |
|:----:|:-------:|:-------:|
| 命中时操作 | `first = mid; right = mid - 1` | `last = mid; left = mid + 1` |
| 搜索方向 | 向左收缩 | 向右收缩 |
| 最终状态 | `first` 为最左 target | `last` 为最右 target |
| 未找到 | 保持 `-1` | 保持 `-1` |

| 整体复杂度 | 时间 $O(\log n)$，空间 $O(1)$ |
|:---------:|:-----------------------------:|

---

> 整理日期：2026-07-31  
> 题目来源：LeetCode 34. Find First and Last Position of Element in Sorted Array
