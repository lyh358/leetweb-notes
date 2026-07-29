# C++ `count()` 完全梳理（力扣刷题版）

## 一、两种 `count()` 的本质区别

在 C++ 中，你遇到的 `count()` 来自**两个不同的来源**，功能和用法完全不同：

| 来源 | 调用方式 | 功能 | 时间复杂度 | 典型容器 |
|------|---------|------|-----------|---------|
| **`<algorithm>` 算法库** | `std::count(begin, end, value)` | 统计值出现次数 | O(n) | vector, string, list, array |
| **关联容器成员函数** | `container.count(key)` | 统计 key 出现次数 | O(1) 或 O(log n) | set, map, unordered_set, unordered_map |

---

## 二、`<algorithm>` 中的 `std::count()`

### 功能
遍历指定范围，统计**等于目标值**的元素个数。

### 原型
```cpp
#include <algorithm>

// 返回值类型通常是 ptrdiff_t 或迭代器差值类型
count(InputIt first, InputIt last, const T& value);
```

### 用法示例
```cpp
vector<int> nums = {1, 2, 2, 3, 2, 4};

// 统计 2 出现了几次
int cnt = count(nums.begin(), nums.end(), 2);  
// cnt = 3

string s = "hello world";
int cnt_h = count(s.begin(), s.end(), 'l');  
// cnt_h = 3
```

### 适用范围
- ✅ **vector, deque, list, string, array, 原生数组**
- ❌ **不能用于 set/map/unordered_set/unordered_map**（它们不是随机访问迭代器，且有自己的 `count()` 成员函数）

---

## 三、关联容器的 `.count()` 成员函数

这是力扣中最常用、最高频的用法，尤其是哈希表相关题目。

### 1. `unordered_set` / `unordered_map`
```cpp
unordered_set<int> us = {1, 3, 5, 7};

// 判断 3 是否在集合中
if (us.count(3)) {        // 返回 1（存在）
    // ...
}

// 判断 4 是否在集合中
if (!us.count(4)) {       // 返回 0（不存在）
    // ...
}
```

### 2. `set` / `map`
```cpp
set<int> s = {1, 3, 5, 7};

// 同样返回 0 或 1
if (s.count(3)) {         // 返回 1
    // ...
}
```

### 3. 带 `multi` 的版本（multiset / multimap）
```cpp
multiset<int> ms = {1, 2, 2, 2, 3};

int cnt = ms.count(2);    // 返回 3（2 出现了 3 次）
```

### 返回值速查表

| 容器类型 | `count(key)` 返回值 | 含义 |
|---------|-------------------|------|
| `unordered_set` / `set` | 0 或 1 | 0=不存在，1=存在 |
| `unordered_map` / `map` | 0 或 1 | 0=key 不存在，1=key 存在 |
| `unordered_multiset` / `multiset` | ≥0 的整数 | key 出现的次数 |
| `unordered_multimap` / `multimap` | ≥0 的整数 | key 出现的次数 |

---

## 四、为什么力扣中更爱用 `.count()` 做存在性判断？

在哈希表题目中，判断"某个元素是否存在"是极其高频的操作。`.count()` 是最简洁的写法：

```cpp
// 写法 1：用 count() 判断存在（最简洁，力扣最常见）
if (uset.count(x)) { ... }

// 写法 2：用 find() 判断存在
if (uset.find(x) != uset.end()) { ... }

// 写法 3：用 contains()（C++20，力扣部分编译器可能不支持）
if (uset.contains(x)) { ... }
```

> 💡 **`.count()` 在 set/map 中判断存在性，本质上等价于 `.find() != end()`**，但代码更短，是力扣题解中的主流写法。

---

## 五、适用范围总结（力扣常见题型）

| 场景 | 推荐用法 | 示例题目 |
|------|---------|---------|
| 判断元素是否在集合中 | `uset.count(x)` | 128. 最长连续序列 |
| 判断 key 是否在字典中 | `umap.count(key)` | 1. 两数之和 |
| 统计数组中某值出现次数 | `count(vec.begin(), vec.end(), x)` | 169. 多数元素 |
| 需要去重 + 快速查找 | `unordered_set` + `.count()` | 217. 存在重复元素 |
| 需要计数 + 快速查找 | `unordered_map` + `.count()` | 347. 前 K 个高频元素 |

---

## 六、一个容易混淆的点

```cpp
unordered_set<int> us;

// ❌ 错误：算法库的 count 不能这样用
count(us, 5);           // 编译错误！

// ✅ 正确：用成员函数
us.count(5);            // 返回 0 或 1

// ✅ 正确：算法库的 count 需要迭代器范围
count(us.begin(), us.end(), 5);   // 也可以，但没必要，且是 O(n)
```

---

## 七、一句话总结

> **`std::count()`** 是算法库函数，用于线性容器遍历统计，O(n)。  
> **`.count()`** 是关联容器成员函数，用于快速判断 key 是否存在（或出现次数），O(1)/O(log n)。  
> 力扣哈希表题目中，**`.count()` 是判断存在的首选写法**。