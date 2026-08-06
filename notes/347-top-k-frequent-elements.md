# 力扣 347. 前 K 个高频元素 — 学习笔记

## 一、题目概述

给定一个非空的整数数组 `nums`，返回其中出现频率前 `k` 高的元素。

> **注意**：可以按任意顺序返回答案。
>
> 进阶：尝试设计时间复杂度优于 $O(n \log n)$ 的算法。

---

## 二、大根堆解法

### 核心思想

1. 先用哈希表统计每个数字的出现频率
2. 用大根堆（按频率降序）存储所有数字
3. 弹出堆顶前 `k` 个元素即为答案

### 代码实现

```cpp
class Solution {
public:
    vector<int> topKFrequent(vector<int>& nums, int k) {
        // 建立哈希表：key-value 对应 num → times
        unordered_map<int, int> occurTimes;
        for (auto num : nums) {
            occurTimes[num]++;  // 创建键值对时值默认为 0
        }

        // 创建大根堆：成员为 pair，对应 map 的键值对
        priority_queue<pair<int, int>> topK;

        for (auto& [num, count] : occurTimes) {
            // 大根堆默认按照 pair.first 来排序
            // 所以插入的时候要反过来，插入 {value, key} = {count, num}
            // 这样堆顶就是频次最高的元素
            topK.push({count, num});
        }

        // 建立数组储存前 K 频次的 num
        vector<int> ans_topK(k);
        for (int i = 0; i < k; i++) {
            ans_topK[i] = topK.top().second;  // 取数字（second 是 num）
            topK.pop();
        }
        return ans_topK;
    }
};
```

### 复杂度分析

| 指标 | 复杂度 | 说明 |
|------|--------|------|
| **时间复杂度** | $O(n \log n)$ | 统计频率 $O(n)$，建堆 $O(n)$，弹出 $k$ 次 $O(k \log n)$ |
| **空间复杂度** | $O(n)$ | 哈希表 $O(n)$，堆 $O(n)$ |

---

## 三、语法详解：为什么 `auto& [num, count]` 用中括号，`{count, num}` 用大括号？

### `auto& [num, count]` —— C++17 结构化绑定（Structured Binding）

```cpp
for (auto& [num, count] : occurTimes) {
```

- **中括号 `[]`** 是 C++17 引入的**结构化绑定**语法
- 作用：把 `map` 的每个 `pair<const Key, T>` **解构**成两个变量
- `num` 对应 `pair.first`（即 key）
- `count` 对应 `pair.second`（即 value）

```cpp
// 等价于下面的传统写法
for (auto& entry : occurTimes) {
    int num = entry.first;      // key
    int count = entry.second;   // value
}
```

> 💡 **注意**：`auto&` 表示引用绑定，避免拷贝。对于 `unordered_map`，迭代器返回的是 `pair<const int, int>&`，所以用 `auto&` 是安全的。

---

### `{count, num}` —— `std::pair` 的列表初始化（List Initialization）

```cpp
topK.push({count, num});
```

- **大括号 `{}`** 是 C++11 引入的**列表初始化**语法
- 这里用来构造一个 `pair<int, int>` 对象
- 等价于：

```cpp
topK.push(make_pair(count, num));
// 或
topK.push(pair<int, int>(count, num));
```

---

### 为什么要"反过来插入"？

```cpp
for (auto& [num, count] : occurTimes) {
    topK.push({count, num});  // 注意：count 在前，num 在后
}
```

**原因：`priority_queue<pair<int, int>>` 默认按 `pair.first` 排序。**

| 存入方式 | `pair` 内容 | 排序依据 | 结果 |
|---------|------------|---------|------|
| `{count, num}` | `first=count, second=num` | 按 `count` 降序 | ✅ 堆顶是频率最高的 |
| `{num, count}` | `first=num, second=count` | 按 `num` 降序 | ❌ 堆顶是数值最大的 |

**`pair` 的默认比较规则（字典序）：**

```cpp
pair<int, int> a = {5, 1};
pair<int, int> b = {3, 100};
// a > b 为 true，因为 5 > 3（先比较 first）
```

而 `priority_queue` 默认是**大根堆**（堆顶是最大元素），所以它会按 `pair` 的默认比较规则把 `first` 大的放上面。

> 因此，**必须把 `count`（频率）放在 `first` 位置**，才能实现"按频率从高到低排序"。

---

## 四、优化解法：小根堆（维护 k 个元素）

### 核心思想

不需要保存所有元素，只需要维护一个大小为 `k` 的小根堆：
- 堆顶是这 `k` 个元素中频率最低的
- 遍历哈希表时，如果当前元素频率比堆顶高，就替换堆顶
- 遍历结束后，堆中保留的就是频率最高的 `k` 个元素

### C++ 代码实现

```cpp
class Solution {
public:
    vector<int> topKFrequent(vector<int>& nums, int k) {
        // 第一步：统计频率
        unordered_map<int, int> freq;
        for (int num : nums) {
            freq[num]++;
        }

        // 第二步：维护大小为 k 的小根堆
        // 自定义比较规则：按频率升序（小根堆，堆顶是频率最小的）
        priority_queue<pair<int, int>, vector<pair<int, int>>, greater<pair<int, int>>> minHeap;

        for (auto& [num, count] : freq) {
            minHeap.push({count, num});
            // 堆的大小超过 k 时，弹出频率最小的那个
            if (minHeap.size() > k) {
                minHeap.pop();
            }
        }

        // 第三步：提取结果
        vector<int> result;
        while (!minHeap.empty()) {
            result.push_back(minHeap.top().second);
            minHeap.pop();
        }

        return result;
    }
};
```

### 复杂度分析

| 指标 | 复杂度 | 说明 |
|------|--------|------|
| **时间复杂度** | $O(n \log k)$ | 每次 push/pop 为 $O(\log k)$，共 $n$ 次 |
| **空间复杂度** | $O(n + k)$ | 哈希表 $O(n)$，堆 $O(k)$ |

> 当 $k \ll n$ 时，这是更优的选择。

---

## 五、进阶解法：桶排序

### 核心思想

利用"频率"这个特征：
- 数字最多出现 `n` 次（数组长度）
- 创建一个"桶数组"，`buckets[i]` 存放所有出现 `i` 次的数字
- 从高频到低频遍历桶，收集前 `k` 个元素

### C++ 代码实现

```cpp
class Solution {
public:
    vector<int> topKFrequent(vector<int>& nums, int k) {
        // 第一步：统计频率
        unordered_map<int, int> freq;
        for (int num : nums) {
            freq[num]++;
        }

        // 第二步：桶排序
        // 下标 = 频率，值 = 具有该频率的所有数字列表
        // 最大频率不超过 nums.size()
        vector<vector<int>> buckets(nums.size() + 1);
        for (auto& [num, count] : freq) {
            buckets[count].push_back(num);
        }

        // 第三步：从高频到低频收集结果
        vector<int> result;
        for (int i = buckets.size() - 1; i >= 0 && result.size() < k; i--) {
            for (int num : buckets[i]) {
                result.push_back(num);
                if (result.size() == k) break;
            }
        }

        return result;
    }
};
```

### 复杂度分析

| 指标 | 复杂度 | 说明 |
|------|--------|------|
| **时间复杂度** | $O(n)$ | 统计频率 $O(n)$，建桶 $O(n)$，收集 $O(n)$ |
| **空间复杂度** | $O(n)$ | 哈希表 + 桶数组 |

> 这是真正的线性时间解法，满足进阶要求。

---

## 六、解法对比总结

| 解法 | 时间复杂度 | 空间复杂度 | 核心思想 | 适用场景 |
|------|-----------|-----------|---------|---------|
| **大根堆** | $O(n \log n)$ | $O(n)$ | 全部入堆后取前 k | 代码简洁，快速 AC |
| **小根堆** | $O(n \log k)$ | $O(n + k)$ | 只维护 k 个高频元素 | $k \ll n$ 时最优 |
| **桶排序（进阶）** | $O(n)$ | $O(n)$ | 按频率分桶，从高频收集 | 追求最优时间复杂度 |

---

## 七、关键收获

1. **结构化绑定 `auto& [a, b]` 是 C++17 语法**，用于快速解构 `pair`/`tuple`/`struct`，让代码更简洁。
2. **列表初始化 `{a, b}` 是 C++11 语法**，用于构造 `pair`、`vector` 等对象，比 `make_pair` 更直观。
3. **`pair` 的默认比较是字典序**：先比较 `first`，`first` 相同再比较 `second`。所以放入优先队列时，**必须把想要排序的字段放在 `first` 位置**。
4. **小根堆优化空间**：大根堆解法虽然正确，但可以优化为小根堆只维护 `k` 个元素，时间从 $O(n \log n)$ 降到 $O(n \log k)$。
5. **桶排序是线性时间的最优解**：利用"频率有上界（不超过 n）"这一特性，用空间换时间达到 $O(n)$。
