# 力扣 215. 数组中的第 K 个最大元素 — 学习笔记

## 一、题目概述

给定整数数组 `nums` 和整数 `k`，请返回数组中第 `k` 个最大的元素。

> **注意**：是排序后的第 `k` 个最大元素，不是第 `k` 个不同的元素。
>
> 例如：`[3, 2, 1, 5, 6, 4]`，排序后为 `[6, 5, 4, 3, 2, 1]`，第 2 个最大元素是 `5`。

---

## 二、大根堆解法（你提供的代码）

### 核心思想

利用 C++ 的 `priority_queue`（默认是大根堆），将所有元素入堆后，堆顶就是全局最大值。连续弹出 `k-1` 次后，堆顶即为第 `k` 大的元素。

### 代码分析

```cpp
class Solution {
public:
    int findKthLargest(vector<int>& nums, int k) {
        // 默认是大根堆（堆顶为最大值）
        priority_queue<int> pq;

        // 将所有元素压入堆中
        for (auto num : nums) {
            pq.push(num);
        }

        // 弹出前 k-1 个最大的元素
        for (int i = 0; i < k - 1; i++) {
            pq.pop();
        }

        // 此时堆顶就是第 k 大的元素
        return pq.top();
    }
};
```

### 复杂度分析

| 指标 | 复杂度 | 说明 |
|------|--------|------|
| **时间复杂度** | $O(n \log n)$ | 建堆 $O(n)$，弹出 $k-1$ 次每次 $O(\log n)$，总体 $O(n \log n)$ |
| **空间复杂度** | $O(n)$ | 堆中存储了所有 $n$ 个元素 |
| **是否修改原数组** | ❌ 否 | 只读遍历 |

### 优缺点

- ✅ 思路直观，代码简洁，利用 STL 现成数据结构
- ❌ 空间 $O(n)$，没有利用"只需要第 k 大"这个信息来优化

---

## 三、优化解法：小根堆（维护 k 个元素）

### 核心思想

不需要保存所有元素，只需要维护一个大小为 `k` 的小根堆：
- 堆顶是这 `k` 个元素中的最小值（即当前第 `k` 大的候选）
- 遍历数组时，如果当前元素比堆顶大，就替换堆顶
- 遍历结束后，堆顶就是第 `k` 大的元素

### 代码实现

```cpp
class Solution {
public:
    int findKthLargest(vector<int>& nums, int k) {
        // 小根堆：堆顶是堆中最小的元素
        priority_queue<int, vector<int>, greater<int>> minHeap;

        for (int num : nums) {
            minHeap.push(num);
            // 堆的大小超过 k 时，弹出最小的那个
            // 这样堆中始终保留的是当前遍历过的元素中最大的 k 个
            if (minHeap.size() > k) {
                minHeap.pop();
            }
        }

        // 堆顶就是最大的 k 个元素中最小的那个，即第 k 大
        return minHeap.top();
    }
};
```

### 复杂度分析

| 指标 | 复杂度 | 说明 |
|------|--------|------|
| **时间复杂度** | $O(n \log k)$ | 每次 push/pop 为 $O(\log k)$，共 $n$ 次 |
| **空间复杂度** | $O(k)$ | 堆中最多保留 $k$ 个元素 |

> 当 $k \ll n$ 时，这是更优的选择。

---

## 四、最优解法：快速选择（Quickselect）

### 核心思想

借鉴快速排序的**分区（partition）**思想：
- 随机选一个 pivot，将数组分为「大于 pivot」和「小于 pivot」两部分
- 如果 pivot 的位置正好是第 `k` 大的位置，直接返回
- 如果 pivot 的位置偏左，说明第 `k` 大的在右边，递归右边
- 如果 pivot 的位置偏右，说明第 `k` 大的在左边，递归左边

平均情况下每次排除一半元素，期望时间复杂度为 $O(n)$。

### 代码实现

```cpp
class Solution {
public:
    int findKthLargest(vector<int>& nums, int k) {
        // 第 k 大 = 第 (n - k) 小（0-based 索引）
        return quickSelect(nums, 0, nums.size() - 1, nums.size() - k);
    }

private:
    int quickSelect(vector<int>& nums, int left, int right, int k) {
        if (left == right) return nums[left];

        // 随机选 pivot，避免最坏情况
        int pivotIndex = left + rand() % (right - left + 1);
        pivotIndex = partition(nums, left, right, pivotIndex);

        if (pivotIndex == k) {
            return nums[k];
        } else if (pivotIndex < k) {
            return quickSelect(nums, pivotIndex + 1, right, k);
        } else {
            return quickSelect(nums, left, pivotIndex - 1, k);
        }
    }

    int partition(vector<int>& nums, int left, int right, int pivotIndex) {
        int pivotValue = nums[pivotIndex];
        // 把 pivot 移到末尾
        swap(nums[pivotIndex], nums[right]);

        int storeIndex = left;
        for (int i = left; i < right; i++) {
            if (nums[i] < pivotValue) {
                swap(nums[i], nums[storeIndex]);
                storeIndex++;
            }
        }

        // 把 pivot 放到正确的位置
        swap(nums[storeIndex], nums[right]);
        return storeIndex;
    }
};
```

### 复杂度分析

| 指标 | 复杂度 | 说明 |
|------|--------|------|
| **平均时间** | $O(n)$ | 每次排除一半元素，$n + n/2 + n/4 + ... < 2n$ |
| **最坏时间** | $O(n^2)$ | 每次 pivot 都选到极值（如已排序数组且总是选第一个） |
| **空间复杂度** | $O(\log n)$ | 递归栈深度 |
| **是否修改原数组** | ✅ 是 | 原地分区 |

> 实际工程中可以通过**随机选 pivot** 或**三数取中法**来避免最坏情况。

---

## 五、其他解法对比

### 方法一：排序

```cpp
class Solution {
public:
    int findKthLargest(vector<int>& nums, int k) {
        sort(nums.begin(), nums.end(), greater<int>());
        return nums[k - 1];
    }
};
```

- **时间**：$O(n \log n)$ | **空间**：$O(\log n)$（排序栈空间）
- ✅ 代码极简，一行搞定
- ❌ 做了多余的排序工作（只需要第 k 大，不需要全部有序）

---

### 方法二：计数排序（适用于数值范围小的情况）

```cpp
class Solution {
public:
    int findKthLargest(vector<int>& nums, int k) {
        // 假设数值范围在 [-10000, 10000]
        vector<int> count(20001, 0);
        for (int num : nums) {
            count[num + 10000]++;
        }

        // 从大到小遍历计数数组
        for (int i = 20000; i >= 0; i--) {
            k -= count[i];
            if (k <= 0) return i - 10000;
        }
        return 0;
    }
};
```

- **时间**：$O(n + V)$（$V$ 为值域大小）| **空间**：$O(V)$
- ✅ 线性时间，当值域很小时非常高效
- ❌ 依赖值域范围，不通用

---

## 六、解法对比总结

| 解法 | 平均时间 | 最坏时间 | 空间复杂度 | 是否修改原数组 | 适用场景 |
|------|---------|---------|-----------|---------------|---------|
| **快速选择（最优）** | $O(n)$ | $O(n^2)$ | $O(\log n)$ | ✅ 是 | 追求最优时间，允许修改数组 |
| 大根堆（你的解法） | $O(n \log n)$ | $O(n \log n)$ | $O(n)$ | ❌ 否 | 代码简洁，不修改数组 |
| 小根堆 | $O(n \log k)$ | $O(n \log k)$ | $O(k)$ | ❌ 否 | $k \ll n$ 时最优 |
| 排序 | $O(n \log n)$ | $O(n \log n)$ | $O(\log n)$ | ✅ 是 | 代码极简，面试快速 AC |
| 计数排序 | $O(n + V)$ | $O(n + V)$ | $O(V)$ | ❌ 否 | 值域很小且已知 |

---

## 七、关键收获

1. **堆解法的优化空间**：你的大根堆解法虽然正确，但可以优化为小根堆维护 `k` 个元素，空间从 $O(n)$ 降到 $O(k)$，时间从 $O(n \log n)$ 降到 $O(n \log k)$。
2. **快速选择是理论最优解**：平均 $O(n)$ 时间找到第 $k$ 大，是面试中的"标准答案"。核心思想是快速排序的 partition，但只递归需要的那一半。
3. **随机选 pivot 的重要性**：如果不随机，已排序数组会让快速选择退化到 $O(n^2)$。
4. **"第 k 大"可以转化为"第 (n-k) 小"**：在 0-based 索引中更方便处理。
5. **排序虽然简单，但不是最优**：面试中如果直接写排序，可能会被追问更优解法。
