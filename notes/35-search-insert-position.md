# LeetCode 35. 搜索插入位置 —— 二分查找学习笔记

> **题目**: [35. Search Insert Position](https://leetcode.cn/problems/search-insert-position/)  
> **核心**: 给定一个**升序排列**的数组 `nums` 和一个目标值 `target`，如果 `target` 在数组中，返回其索引；如果不在，返回它**按顺序插入的位置**（保证数组仍有序）。  
> **示例**:  
> ```
> nums = [1, 3, 5, 6], target = 5  → 返回 2（已存在，返回索引）
> nums = [1, 3, 5, 6], target = 2  → 返回 1（应插入到 1 和 3 之间）
> nums = [1, 3, 5, 6], target = 7  → 返回 4（应插入到末尾）
> nums = [1, 3, 5, 6], target = 0  → 返回 0（应插入到开头）
> ```

---

## 一、核心思想：二分查找定位插入点

### 1.1 问题本质
这道题是二分查找的**经典变式**：不仅要查找目标值是否存在，还要在**不存在时返回它应该插入的位置**，使得数组保持有序。

换句话说，我们要找的是：
> **数组中第一个大于或等于 target 的元素的位置**（即 lower_bound）

- 如果数组中有等于 target 的元素，返回它的索引；
- 如果没有，返回第一个比 target 大的元素的索引（即插入位置）。

### 1.2 为什么用二分查找？
数组是升序的，满足二分查找的前提条件。线性扫描需要 $O(n)$，而二分查找只需 $O(\log n)$。

---

## 二、代码实现（`while (left <= right)` 闭区间写法）

```java
class Solution {
    public int searchInsert(int[] nums, int target) {
        // ========== 初始化搜索边界 ==========
        // 采用 [left, right] 闭区间，两端都是合法下标
        int left = 0;
        int right = nums.length - 1;    // right 指向最后一个元素

        // ========== 二分查找主循环 ==========
        // 循环条件：left <= right
        // 含义：当搜索范围内还有元素时，继续查找
        while (left <= right) {
            // 计算中点，防溢出写法
            int mid = left + (right - left) / 2;

            if (nums[mid] == target) {
                // 找到目标值，直接返回其索引
                return mid;
            } else if (nums[mid] > target) {
                // 中间值比目标大：target 只可能在左半部分
                // 将右边界移到 mid - 1，排除 mid 及其右侧所有元素
                right = mid - 1;
            } else {
                // 中间值比目标小：target 只可能在右半部分
                // 将左边界移到 mid + 1，排除 mid 及其左侧所有元素
                left = mid + 1;
            }
        }

        // ========== 循环结束，返回插入位置 ==========
        // 此时 left > right，搜索范围为空
        // left 指向第一个大于 target 的位置（或数组末尾的下一个位置）
        // 这个位置就是 target 应该插入的位置
        return left;
    }
}
```

### 2.1 为什么循环结束后 `return left` 就是插入位置？

循环结束时，`left > right`，此时：
- `right` 指向最后一个小于 target 的元素（或 -1）
- `left` 指向第一个大于 target 的元素（或 nums.length）

而**第一个大于 target 的位置**，正是 target 应该插入的位置！

```
数组: [1, 3, 5, 6], target = 2

查找过程把范围缩到空：
left = 1, right = 0（left > right，循环结束）

此时：
- right = 0，nums[0] = 1 < 2（最后一个小于 target 的）
- left = 1，nums[1] = 3 > 2（第一个大于 target 的）

所以 target = 2 应该插入到索引 1 的位置 ✓
```

---

## 三、逐步推演示例

### 示例 1：目标值在数组中
```
nums = [1, 3, 5, 6], target = 5
```

| 轮次 | left | right | mid | nums[mid] | 比较 | 操作 | 新区间 |
|:----:|:----:|:-----:|:---:|:---------:|:----:|:----:|:------:|
| 初始 | 0 | 3 | — | — | — | — | [0, 3] |
| 1 | 0 | 3 | 1 | nums[1]=3 | 5 > 3 | left = 2 | [2, 3] |
| 2 | 2 | 3 | 2 | nums[2]=5 | 5 == 5 | **返回 2** | — |

**结果**: `2` ✓

---

### 示例 2：目标值不在数组中（插入中间）
```
nums = [1, 3, 5, 6], target = 2
```

| 轮次 | left | right | mid | nums[mid] | 比较 | 操作 | 新区间 |
|:----:|:----:|:-----:|:---:|:---------:|:----:|:----:|:------:|
| 初始 | 0 | 3 | — | — | — | — | [0, 3] |
| 1 | 0 | 3 | 1 | nums[1]=3 | 2 < 3 | right = 0 | [0, 0] |
| 2 | 0 | 0 | 0 | nums[0]=1 | 2 > 1 | left = 1 | [1, 0] |
| — | 1 | 0 | — | — | left > right，循环结束 | — | — |

**循环结束后**: `left = 1`, `right = 0`

`left = 1` 指向第一个大于 2 的元素（nums[1] = 3），即插入位置。

**结果**: `1` ✓

---

### 示例 3：目标值大于所有元素（插入末尾）
```
nums = [1, 3, 5, 6], target = 7
```

| 轮次 | left | right | mid | nums[mid] | 比较 | 操作 | 新区间 |
|:----:|:----:|:-----:|:---:|:---------:|:----:|:----:|:------:|
| 初始 | 0 | 3 | — | — | — | — | [0, 3] |
| 1 | 0 | 3 | 1 | nums[1]=3 | 7 > 3 | left = 2 | [2, 3] |
| 2 | 2 | 3 | 2 | nums[2]=5 | 7 > 5 | left = 3 | [3, 3] |
| 3 | 3 | 3 | 3 | nums[3]=6 | 7 > 6 | left = 4 | [4, 3] |
| — | 4 | 3 | — | — | left > right，循环结束 | — | — |

**循环结束后**: `left = 4`, `right = 3`

`left = 4` 等于数组长度，表示 target 应插入到数组末尾。

**结果**: `4` ✓

---

### 示例 4：目标值小于所有元素（插入开头）
```
nums = [1, 3, 5, 6], target = 0
```

| 轮次 | left | right | mid | nums[mid] | 比较 | 操作 | 新区间 |
|:----:|:----:|:-----:|:---:|:---------:|:----:|:----:|:------:|
| 初始 | 0 | 3 | — | — | — | — | [0, 3] |
| 1 | 0 | 3 | 1 | nums[1]=3 | 0 < 3 | right = 0 | [0, 0] |
| 2 | 0 | 0 | 0 | nums[0]=1 | 0 < 1 | right = -1 | [0, -1] |
| — | 0 | -1 | — | — | left > right，循环结束 | — | — |

**循环结束后**: `left = 0`, `right = -1`

`left = 0` 指向数组开头，表示 target 应插入到最前面。

**结果**: `0` ✓

---

### 示例 5：单元素数组
```
nums = [1], target = 0
```

| 轮次 | left | right | mid | nums[mid] | 比较 | 操作 | 新区间 |
|:----:|:----:|:-----:|:---:|:---------:|:----:|:----:|:------:|
| 初始 | 0 | 0 | — | — | — | — | [0, 0] |
| 1 | 0 | 0 | 0 | nums[0]=1 | 0 < 1 | right = -1 | [0, -1] |
| — | 0 | -1 | — | — | left > right，循环结束 | — | — |

**结果**: `left = 0` → 返回 `0` ✓

```
nums = [1], target = 2
```

| 轮次 | left | right | mid | nums[mid] | 比较 | 操作 | 新区间 |
|:----:|:----:|:-----:|:---:|:---------:|:----:|:----:|:------:|
| 初始 | 0 | 0 | — | — | — | — | [0, 0] |
| 1 | 0 | 0 | 0 | nums[0]=1 | 2 > 1 | left = 1 | [1, 0] |
| — | 1 | 0 | — | — | left > right，循环结束 | — | — |

**结果**: `left = 1` → 返回 `1` ✓

---

## 四、关键机制深度解析

### 4.1 闭区间 `[left, right]` 的含义

```
left = 0, right = nums.length - 1
```

- `left` 和 `right` 都是**合法下标**，都指向数组中的真实元素；
- 搜索范围是**两端都闭合**的区间 `[left, right]`；
- 循环条件 `left <= right` 表示：只要范围内还有元素，就继续查找。

### 4.2 边界更新的正确性

```java
if (nums[mid] > target)  right = mid - 1;   // ✅ 排除 mid 及其右侧
if (nums[mid] < target)  left = mid + 1;    // ✅ 排除 mid 及其左侧
```

**为什么 `right = mid - 1`？**
- `nums[mid] > target` 说明 `mid` 位置的元素已经比 target 大了；
- 由于数组升序，`mid` 右侧的所有元素都 `>= nums[mid] > target`；
- 所以 `mid` 及其右侧都不可能包含 target，全部排除。

**为什么 `left = mid + 1`？**
- `nums[mid] < target` 说明 `mid` 位置的元素已经比 target 小了；
- 由于数组升序，`mid` 左侧的所有元素都 `<= nums[mid] < target`；
- 所以 `mid` 及其左侧都不可能包含 target，全部排除。

### 4.3 循环结束后 `left` 为什么就是插入位置？

循环结束时，`left > right`，此时搜索范围为空。但 `left` 和 `right` 的位置有明确含义：

```
数组: [ ... , nums[right], nums[left], ... ]
                ↑           ↑
             最后一个小于   第一个大于等于
             target 的元素  target 的元素
```

- `right` 指向**最后一个小于 target** 的元素（或 -1）；
- `left` 指向**第一个大于等于 target** 的元素（或 nums.length）。

而**第一个大于等于 target 的位置**，正是 lower_bound，也就是 target 应该插入的位置！

### 4.4 找到 target 时为什么可以直接 `return mid`？

因为题目只要求返回 target 的**任意一个**索引即可。如果数组中有多个相同的 target，返回哪一个都可以。

如果题目要求返回**最左边**的 target，则需要修改代码（见下方扩展部分）。

---

## 五、与 `while (left < right)` 写法的对比

| 对比项 | `while (left <= right)` 闭区间 ⭐ | `while (left < right)` 左闭右开 |
|:------:|:--------------------------------:|:-------------------------------:|
| **初始化** | `left=0, right=n-1` | `left=0, right=n` |
| **区间含义** | `[left, right]` 两端都合法 | `[left, right)`，right 是哨兵 |
| **循环条件** | `<=`：范围内有元素就继续 | `<`：至少两个位置才继续 |
| **边界更新（大）** | `right = mid - 1` | `right = mid` |
| **边界更新（小）** | `left = mid + 1` | `left = mid + 1` |
| **终止状态** | `left > right` | `left == right` |
| **返回值** | `left`（lower_bound） | `left`（lower_bound） |
| **找到时** | 可立即 `return mid` | 需等循环结束，或额外处理 |
| **代码直观度** | ⭐⭐⭐ 非常直观 | ⭐⭐ 需理解开区间 |
| **推荐场景** | 初学者首选，面试推荐 | 找最左/最右重复元素 |

### 为什么推荐 `left <= right` 作为入门写法？

1. **最符合直觉**：`left` 和 `right` 都是下标，都指向真实元素；
2. **边界更新简单**：查过的 `mid` 直接排除（`mid ± 1`），不会重复检查；
3. **容易验证**：手推示例时，区间变化清晰明了；
4. **与教材一致**：大多数算法教材和面试题解都采用此写法。

---

## 六、复杂度分析

| 维度 | 结果 |
|------|------|
| **时间复杂度** | $O(\log n)$ —— 二分查找，每次范围减半 |
| **空间复杂度** | $O(1)$ —— 只使用了 `left, right, mid` 三个变量 |

---

## 七、易错点与面试 Tips

### ❌ 错误 1：循环条件写成 `left < right`
```java
// 如果初始化是 left=0, right=n-1，但用 <，会漏掉 left==right 的情况
while (left < right) {  // ❌ 与闭区间初始化不匹配
```

### ❌ 错误 2：边界更新写成 `right = mid`
```java
if (nums[mid] > target) {
    right = mid;  // ❌ 错误！mid 已确认 > target，应排除
}
// 正确写法
right = mid - 1;  // ✅
```

### ❌ 错误 3：返回值写成 `right`
```java
return right;  // ❌ 错误！right 指向最后一个小于 target 的元素
return left;   // ✅ 正确！left 指向第一个大于等于 target 的位置
```

### ❌ 错误 4：`mid` 计算溢出
```java
int mid = (left + right) / 2;  // 不推荐，有溢出风险
int mid = left + (right - left) / 2;  // ✅ 推荐
```

### ❌ 错误 5：数组为空或长度为 0 未处理
```java
// 本题 nums 长度 >= 1，但如果需要更严谨：
if (nums == null || nums.length == 0) return 0;
```

### ✅ 面试回答框架
1. **思路**：利用数组升序特性，二分查找 target；
2. **区间定义**：`[left, right]` 闭区间，`left=0, right=n-1`；
3. **查找逻辑**：等于则返回，大于则缩右边界，小于则缩左边界；
4. **插入位置**：若未找到，循环结束时 `left` 即为 lower_bound，就是插入位置；
5. **复杂度**：时间 $O(\log n)$，空间 $O(1)$。

### ✅ 追问准备
- **问**：如果有多个相同的 target，返回哪个位置？  
  **答**：本题代码返回找到的任意一个。若要返回最左边的，需将 `nums[mid] == target` 时的处理改为 `right = mid - 1`，循环结束后再判断。

- **问**：数组是降序的怎么办？  
  **答**：调整比较方向，或先对数组进行反转/取反处理。

---

## 八、扩展：查找最左边/最右边的 target

### 查找最左边的 target（lower_bound 严格版）

```java
public int searchLeft(int[] nums, int target) {
    int left = 0, right = nums.length - 1;

    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] >= target) {
            right = mid - 1;  // 即使相等也向左缩，找更左边的
        } else {
            left = mid + 1;
        }
    }

    // 循环结束后，left 指向第一个 >= target 的位置
    // 需检查是否越界以及是否等于 target
    if (left < nums.length && nums[left] == target) {
        return left;
    }
    return left;  // 或返回 -1 表示不存在
}
```

### 查找最右边的 target（upper_bound - 1）

```java
public int searchRight(int[] nums, int target) {
    int left = 0, right = nums.length - 1;

    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] <= target) {
            left = mid + 1;  // 即使相等也向右缩，找更右边的
        } else {
            right = mid - 1;
        }
    }

    // 循环结束后，right 指向最后一个小于等于 target 的位置
    if (right >= 0 && nums[right] == target) {
        return right;
    }
    return -1;
}
```

---

## 九、一句话总结

> **闭区间 `[left, right]` 两头堵，找到 target 直接返；没找到时循环自然结束，`left` 正好停在第一个大于 target 的位置——那就是该插的地方。**

| 要点 | 内容 |
|------|------|
| 区间定义 | `[left, right]` 闭区间，`left=0, right=n-1` |
| 循环条件 | `left <= right` |
| 边界更新 | `right = mid - 1`（排除 mid 及右侧），`left = mid + 1`（排除 mid 及左侧） |
| 找到 target | `return mid` |
| 未找到 target | `return left`（lower_bound，即插入位置） |
| 时间复杂度 | $O(\log n)$ |
| 空间复杂度 | $O(1)$ |

---

> 整理日期：2026-07-31  
> 题目来源：LeetCode 35. Search Insert Position
