# 力扣 283. 移动零 — 踩坑笔记

## 题目信息

- **来源**: LeetCode 283
- **难度**: 简单
- **核心考点**: 双指针、数组原地修改

## 题目大意

给定一个数组 `nums`，将所有 `0` 移动到数组末尾，同时保持非零元素的相对顺序。

要求**必须在原数组上操作**，不能复制额外数组。

---

## 我的原始代码

```cpp
class Solution {
public:
    void moveZeroes(vector<int>& nums) {
        int left = 0, right = 1;
        while (right < nums.size()) {
            while (nums[left] != 0) {   // ❌ 没有越界检查
                left++;
            }
            while (nums[right] == 0) {  // ❌ 没有越界检查
                right++;
            }
            nums[left] = nums[right];   // ❌ 应该用 swap
            nums[right] = 0;
        }
        return;
    }
};
```

---

## 踩坑分析

### 🕳️ 坑 1：内层 while 没有越界检查（最致命）

```cpp
while (nums[left] != 0) {
    left++;     // 如果数组中没有 0，left 会一直递增直到越界！
}
```

**触发场景**：当数组**全为非零元素**时，例如 `nums = [1, 2, 3, 4]`

| 步骤 | left | nums[left] | 操作 |
|------|------|-----------|------|
| 1 | 0 | 1 ≠ 0 | left++ → 1 |
| 2 | 1 | 2 ≠ 0 | left++ → 2 |
| 3 | 2 | 3 ≠ 0 | left++ → 3 |
| 4 | 3 | 4 ≠ 0 | left++ → 4 |
| 5 | 4 | **越界访问！** | 💥 Runtime Error |

> 同理，`while (nums[right] == 0)` 在数组全为 0 时也会导致 right 越界。

**修复**：内层循环必须加上边界判断

```cpp
while (left < nums.size() && nums[left] != 0) left++;
while (right < nums.size() && nums[right] == 0) right++;
```

---

### 🕳️ 坑 2：left 和 right 的初始化与关系没处理好

原始代码中 `left=0, right=1` 的设定过于死板，没有考虑以下情况：

- `left` 找 0 的过程中可能**超过 right**
- `right` 找非 0 的过程中可能**超过数组边界**
- 交换后没有正确推进指针，导致重复交换或死循环

**修复**：每次找完 left 后，right 应该从 left 开始找，而不是固定从 1 开始。

---

### 🕳️ 坑 3：交换逻辑不规范

```cpp
nums[left] = nums[right];
nums[right] = 0;
```

这种手动赋值在这个题目里**碰巧能 work**，但存在隐患：

1. **可读性差**：一眼看不出是在"交换"
2. **隐含假设**：要求 `nums[right]` 原本非 0，且 `left` 和 `right` 之间的元素都是 0
3. **如果 left == right**：会把非 0 元素覆盖成 0，导致错误

**修复**：使用 `swap()`，语义清晰，且能安全处理 left == right 的情况

```cpp
swap(nums[left], nums[right]);
```

---

### 🕳️ 坑 4：循环结束后没有处理剩余元素

原始代码的交换逻辑假设每次都能找到一对 (0, 非0) 进行交换，但如果：
- 找到 left（0的位置）后，right 越界了
- 此时应该直接 break，但原始代码没有处理

---

## 修正后的代码（基于我的思路）

```cpp
class Solution {
public:
    void moveZeroes(vector<int>& nums) {
        int left = 0;

        while (left < nums.size()) {
            // 1. 找左边第一个 0（必须加边界检查）
            while (left < nums.size() && nums[left] != 0) {
                left++;
            }

            // 如果 left 已经越界，说明没有 0 了
            if (left >= nums.size()) break;

            // 2. 从 left 右边找第一个非 0
            int right = left + 1;
            while (right < nums.size() && nums[right] == 0) {
                right++;
            }

            // 如果 right 越界，说明后面全是 0，结束
            if (right >= nums.size()) break;

            // 3. 交换（用 swap 更安全）
            swap(nums[left], nums[right]);

            // 4. left 前进一位，继续下一轮
            left++;
        }
    }
};
```

---

## 更优雅的双指针写法（推荐）

上面的写法虽然能跑，但过于复杂。这道题有一个非常简洁的标准解法：

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

### 为什么这个写法更好？

| 对比项 | 我的写法 | 标准写法 |
|--------|---------|---------|
| 代码行数 | 多，嵌套复杂 | 少，逻辑清晰 |
| 边界处理 | 需要多处判断 | for 循环自带边界 |
| 指针关系 | left 和 right 关系复杂 | right 一直走，left 按需走 |
| 可读性 | 需要理解"找0找非0交换" | 一句话：非零元素往前填 |

### 核心思想

> `left` 指向**当前已经处理好的非零序列的末尾**（也就是下一个非零元素该放的位置）。
> 
> `right` 负责遍历数组，遇到非零元素就和 `left` 交换，然后 `left` 前进一步。

**推演示例**：`nums = [0, 1, 0, 3, 12]`

| right | nums[right] | 操作 | 数组状态 | left |
|-------|------------|------|---------|------|
| 0 | 0 | 跳过 | [0, 1, 0, 3, 12] | 0 |
| 1 | 1 | swap(nums[0], nums[1]), left++ | [1, 0, 0, 3, 12] | 1 |
| 2 | 0 | 跳过 | [1, 0, 0, 3, 12] | 1 |
| 3 | 3 | swap(nums[1], nums[3]), left++ | [1, 3, 0, 0, 12] | 2 |
| 4 | 12 | swap(nums[2], nums[4]), left++ | [1, 3, 12, 0, 0] | 3 |

---

## 关键教训

### 1. while 循环必须考虑越界
```cpp
// ❌ 危险
while (nums[i] != 0) i++;

// ✅ 安全
while (i < n && nums[i] != 0) i++;
```

### 2. 交换用 swap，不要手动赋值
```cpp
// ❌ 语义不清，有隐患
a = b; b = 0;

// ✅ 语义清晰，安全
swap(a, b);
```

### 3. 双指针题，先想清楚指针的"职责"
- `left` 负责什么？`right` 负责什么？
- 两个指针是**同向移动**还是**相向移动**？
- 这道题是同向双指针，`right` 探索，`left` 占位

### 4. 简单题也要考虑边界
- 数组长度为 0
- 数组全为 0
- 数组全为非 0
- 只有一个元素

---

## 类似双指针题目推荐

| 题目 | 链接 | 类型 |
|------|------|------|
| 26. 删除有序数组中的重复项 | [LeetCode](https://leetcode.cn/problems/remove-duplicates-from-sorted-array/) | 快慢指针 |
| 27. 移除元素 | [LeetCode](https://leetcode.cn/problems/remove-element/) | 快慢指针 |
| 80. 删除有序数组中的重复项 II | [LeetCode](https://leetcode.cn/problems/remove-duplicates-from-sorted-array-ii/) | 快慢指针 |
| 167. 两数之和 II | [LeetCode](https://leetcode.cn/problems/two-sum-ii-input-array-is-sorted/) | 相向双指针 |
| 344. 反转字符串 | [LeetCode](https://leetcode.cn/problems/reverse-string/) | 相向双指针 |
