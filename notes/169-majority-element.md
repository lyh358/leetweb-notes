# 力扣 169. 多数元素 — 学习笔记（Boyer-Moore 投票法）

## 一、题目概述

给定一个大小为 `n` 的数组 `nums`，返回其中的**多数元素**。

> **多数元素**：在数组中出现次数 **严格大于** `⌊n / 2⌋` 的元素。
>
> **题目保证**：数组非空，且**多数元素一定存在**。

---

## 二、最优解法：Boyer-Moore 投票算法

### 核心思想

把数组中的元素想象成**不同阵营的士兵**在互相厮杀：

- **相同阵营**的士兵相遇，**联手**（票数 +1）
- **不同阵营**的士兵相遇，**同归于尽**（票数 -1）
- 当一个阵营的士兵全部阵亡（票数归零），**新的士兵**接管战场

由于多数元素出现次数 **> n/2**，即使它和其他所有不同元素一一抵消，最终也一定会**存活下来**。

### 代码分析

```cpp
class Solution {
public:
    int majorityElement(vector<int>& nums) 
    {   
        // 投票法：把众数记为 +1，把其他数记为 −1，将它们全部加起来，显然和大于 0

        // 初始化第一个元素为候选众数
        int candidate = nums[0];
        // 既然目前视为是众数，那把（全局）投票 +1
        int vote = 1;

        // 从第二个数（i=1）开始遍历
        for(int i = 1; i < nums.size(); i++)
        {
            // 当目前的数还没被淘汰（还有票）
            if(vote > 0)
            {
                // 如果 x 与 candidate 相等，那么计数器 count 的值增加 1
                if(candidate == nums[i])
                {
                    vote++;
                }
                // 如果 x 与 candidate 不等，那么计数器 count 的值减少 1
                else
                {
                    vote--;
                }
            }
            // 如果当前数的票数 = 0 了，说明它现在不是众数，
            // 让下一个被遍历的数继承众数之位并初始化票数 = 1
            else
            {
                candidate = nums[i];
                vote = 1;
            }
        }
        // 在遍历完成后，candidate 即为整个数组的众数。
        return candidate;
    }
};
```

### 复杂度分析

| 指标 | 复杂度 | 说明 |
|------|--------|------|
| **时间复杂度** | $O(n)$ | 只需一次线性遍历 |
| **空间复杂度** | $O(1)$ | 仅使用两个变量 `candidate` 和 `vote` |
| **是否修改原数组** | ❌ 否 | 只读遍历 |

---

## 三、算法正确性证明（直观理解）

假设多数元素为 `M`，出现次数为 `k`（$k > n/2$）。

将数组中的元素分成两类：
- **M 阵营**：共 `k` 个
- **非 M 阵营**：共 `n - k` 个（$n - k < k$）

在投票过程中：
1. 当遇到 **M** 时，`vote++`
2. 当遇到 **非 M** 时，`vote--`

最坏情况下，每一个 **M** 都和一个 **非 M** 配对抵消。但由于 $k > n - k$，抵消完后 **M 还有剩余**，最终 `candidate` 一定会变成 `M`。

> 💡 **关键前提**：题目已保证多数元素存在。如果去掉这个保证，投票结束后还需要**二次遍历验证** `candidate` 的出现次数是否确实 > n/2。

---

## 四、其他解法对比

### 方法一：哈希表计数

```cpp
class Solution {
public:
    int majorityElement(vector<int>& nums) {
        unordered_map<int, int> count;
        for (int num : nums) {
            if (++count[num] > nums.size() / 2) {
                return num;
            }
        }
        return -1;
    }
};
```

- **时间**：$O(n)$ | **空间**：$O(n)$
- ✅ 思路直观，可扩展（如求出现次数 > n/3 的元素）
- ❌ 需要额外空间

---

### 方法二：排序

```cpp
class Solution {
public:
    int majorityElement(vector<int>& nums) {
        sort(nums.begin(), nums.end());
        return nums[nums.size() / 2];  // 中间位置一定是多数元素
    }
};
```

- **时间**：$O(n \log n)$ | **空间**：$O(\log n)$（排序栈空间）
- ✅ 代码极简，一行核心逻辑
- ❌ 修改了原数组；时间复杂度非最优

> 原理：若某元素出现次数 > n/2，排序后它必然占据数组的**中间位置**。

---

### 方法三：分治法

```cpp
class Solution {
public:
    int majorityElement(vector<int>& nums) {
        return majority(nums, 0, nums.size() - 1);
    }

private:
    int majority(vector<int>& nums, int lo, int hi) {
        if (lo == hi) return nums[lo];

        int mid = lo + (hi - lo) / 2;
        int left = majority(nums, lo, mid);
        int right = majority(nums, mid + 1, hi);

        if (left == right) return left;

        // 统计两个候选人在当前区间的出现次数
        int leftCount = countInRange(nums, left, lo, hi);
        int rightCount = countInRange(nums, right, lo, hi);

        return leftCount > rightCount ? left : right;
    }

    int countInRange(vector<int>& nums, int num, int lo, int hi) {
        int count = 0;
        for (int i = lo; i <= hi; i++) {
            if (nums[i] == num) count++;
        }
        return count;
    }
};
```

- **时间**：$O(n \log n)$ | **空间**：$O(\log n)$（递归栈）
- 核心思想：如果 `a` 是左半区间的多数元素，`b` 是右半区间的多数元素，那么整个区间的多数元素只能是 `a` 或 `b` 之一。

---

### 方法四：随机化

```cpp
class Solution {
public:
    int majorityElement(vector<int>& nums) {
        while (true) {
            int candidate = nums[rand() % nums.size()];
            int count = 0;
            for (int num : nums) {
                if (num == candidate) count++;
            }
            if (count > nums.size() / 2) return candidate;
        }
    }
};
```

- **期望时间**：$O(n)$ | **空间**：$O(1)$
- 原理：由于多数元素占比 > 50%，随机选取一个元素，**期望只需 2 次**就能命中多数元素。

---

## 五、解法对比总结

| 解法 | 时间复杂度 | 空间复杂度 | 是否修改原数组 | 核心思想 |
|------|-----------|-----------|---------------|---------|
| **投票法（最优）** | $O(n)$ | $O(1)$ | ❌ 否 | 不同元素互相抵消 |
| 哈希表计数 | $O(n)$ | $O(n)$ | ❌ 否 | 统计出现频率 |
| 排序 | $O(n \log n)$ | $O(\log n)$ | ✅ 是 | 多数元素必在中间 |
| 分治法 | $O(n \log n)$ | $O(\log n)$ | ❌ 否 | 区间多数元素的合并 |
| 随机化 | 期望 $O(n)$ | $O(1)$ | ❌ 否 | 利用概率优势随机采样 |

---

## 六、关键收获

1. **投票法是处理"多数元素"问题的标准解法**。只要看到"出现次数 > n/2"，第一反应就是 Boyer-Moore 投票。
2. **抵消思想**非常巧妙：不需要知道具体出现了多少次，只需要知道"谁活到了最后"。
3. **注意前提条件**：投票法**依赖"多数元素一定存在"**。如果题目没有这个保证，必须二次验证。
4. **扩展思考**：如果题目改为"找出所有出现次数严格大于 n/3 的元素"，投票法需要维护 **2 个候选人**（因为最多只有 2 个这样的元素）。这就是力扣 **229. 多数元素 II**。

---

## 七、投票法流程图解（示例）

数组：`[2, 2, 1, 1, 1, 2, 2]`，多数元素为 `2`

| 步骤 | 当前元素 | candidate | vote | 说明 |
|------|---------|-----------|------|------|
| 初始 | — | 2 | 1 | 初始化 |
| 1 | 2 | 2 | 2 | 同伙，vote++ |
| 2 | 1 | 2 | 1 | 敌人，vote-- |
| 3 | 1 | 2 | 0 | 敌人，vote-- |
| 4 | 1 | **1** | **1** | vote归零，换候选人 |
| 5 | 2 | 1 | 0 | 敌人，vote-- |
| 6 | 2 | **2** | **1** | vote归零，换候选人 |

最终 `candidate = 2`，即为多数元素。✅
