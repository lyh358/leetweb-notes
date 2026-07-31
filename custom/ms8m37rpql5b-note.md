# 二分查找基础：查询数组中是否存在某个数 —— 学习笔记

> **题目**: P14085.【二分1】查询是否存在某个数  
> **核心**: 给定长度为 $n$ 的**升序**数组，进行 $m$ 次查询，判断每次查询的数是否在数组中。  
> **要求**: 必须使用**二分查找**完成查询。  
> **数据范围**: $1 \leq n, m \leq 10^5$，$|a[i]|, |x| \leq 10^9$

---

## 一、核心思想：二分查找（Binary Search）

### 1.1 为什么不用线性查找？
如果每次查询都从头到尾扫描数组，时间复杂度为 $O(n)$，$m$ 次查询总时间为 $O(nm)$。当 $n = m = 10^5$ 时，总操作量达 $10^{10}$，远超时间限制。

### 1.2 二分查找的原理
数组是**升序排列**的，这是使用二分查找的**前提条件**。利用有序性，每次将搜索范围缩小一半：

```
在 [left, right] 范围内查找 target：
1. 取中间元素 mid
2. 如果 target == a[mid] → 找到了！
3. 如果 target < a[mid] → target 只可能在左半部分，right = mid - 1
4. 如果 target > a[mid] → target 只可能在右半部分，left = mid + 1
5. 重复直到 left > right（范围为空，说明不存在）
```

每次比较后，搜索范围至少减少一半，因此最多只需要 $\log_2 n$ 次比较。

### 1.3 时间复杂度对比

| 方法 | 单次查询 | $m$ 次查询总时间 | 适用条件 |
|:----:|:-------:|:---------------:|:--------:|
| **线性查找** | $O(n)$ | $O(nm)$ | 无序数组 |
| **二分查找** | $O(\log n)$ | $O(m \log n)$ | **升序/降序数组** |
| **哈希表** | $O(1)$ | $O(n + m)$ | 允许额外空间 |

对于 $n = 10^5$，$\log_2 n \approx 17$，二分查找只需约 17 次比较，而线性查找需要 10 万次。

---

## 二、代码实现（带完整注释）

```cpp
#include <iostream>
#include <vector>
using namespace std;

/**
 * 二分查找函数：在升序数组 v 中查找 target
 * 如果存在，输出 "YES" 并返回
 * 如果不存在，输出 "NO" 并返回
 */
void search(vector<int>& v, int target)
{
    // ========== 初始化搜索边界 ==========
    // left 指向当前搜索范围的左端（闭区间）
    // right 指向当前搜索范围的右端（闭区间）
    int left = 0;
    int right = v.size() - 1;

    // ========== 二分查找主循环 ==========
    // 循环条件：left <= right
    // 含义：当左边界不超过右边界时，搜索范围内还有元素待检查
    while (left <= right)
    {
        // ========== 计算中点 ==========
        // 使用 left + (right - left) / 2 而不是 (left + right) / 2
        // 原因：防止 left + right 溢出（虽然本题数据范围不会溢出，但这是良好习惯）
        int mid = left + (right - left) / 2;

        // ========== 比较并缩小范围 ==========
        if (target < v[mid]) {
            // target 比中间值小：说明 target 只可能在左半部分
            // 将右边界移到 mid - 1，排除 mid 及其右侧所有元素
            right = mid - 1;
        }
        else if (target > v[mid]) {
            // target 比中间值大：说明 target 只可能在右半部分
            // 将左边界移到 mid + 1，排除 mid 及其左侧所有元素
            left = mid + 1;
        }
        else {
            // target == v[mid]：找到了！
            cout << "YES" << endl;
            return;  // 直接返回，结束本次查询
        }
    }

    // ========== 循环结束，未找到 ==========
    // 当 left > right 时，说明搜索范围已经为空
    // target 不在数组中
    cout << "NO" << endl;
    return;
}

int main()
{
    // ========== 读入数据 ==========
    int n, Q;           // n = 数组长度, Q = 查询次数（题目用 m，代码用 Q）
    cin >> n >> Q;

    vector<int> v(n);   // 存储升序数组
    vector<int> m(Q);   // 存储所有查询的 target 值（注意：变量名 m 和题目中的查询次数 m 同名，但不影响）

    // 读入数组元素
    for (int i = 0; i < n; i++) {
        cin >> v[i];
    }

    // 读入所有查询值
    for (int i = 0; i < Q; i++) {
        cin >> m[i];
    }

    // ========== 逐个查询 ==========
    for (int i = 0; i < Q; i++) {
        search(v, m[i]);  // 对第 i 个查询值进行二分查找
    }

    return 0;
}
```

---

## 三、逐步推演示例

### 示例：数组 `[1, 3, 5, 7, 9]`，查询 `target = 3`

**初始**: `left = 0`, `right = 4`（数组下标范围 `[0, 4]`）

| 轮次 | left | right | mid | v[mid] | target vs v[mid] | 操作 | 新范围 |
|:----:|:----:|:-----:|:---:|:------:|:----------------:|:----:|:------:|
| 1 | 0 | 4 | 2 | v[2]=5 | 3 < 5 | `right = 1` | [0, 1] |
| 2 | 0 | 1 | 0 | v[0]=1 | 3 > 1 | `left = 1` | [1, 1] |
| 3 | 1 | 1 | 1 | v[1]=3 | 3 == 3 | **找到！** | — |

**结果**: 输出 `YES` ✓

---

### 示例：数组 `[1, 3, 5, 7, 9]`，查询 `target = 4`

**初始**: `left = 0`, `right = 4`

| 轮次 | left | right | mid | v[mid] | target vs v[mid] | 操作 | 新范围 |
|:----:|:----:|:-----:|:---:|:------:|:----------------:|:----:|:------:|
| 1 | 0 | 4 | 2 | v[2]=5 | 4 < 5 | `right = 1` | [0, 1] |
| 2 | 0 | 1 | 0 | v[0]=1 | 4 > 1 | `left = 1` | [1, 1] |
| 3 | 1 | 1 | 1 | v[1]=3 | 4 > 3 | `left = 2` | [2, 1] |

**第 3 轮后**: `left = 2`, `right = 1`，`left > right`，循环结束。

**结果**: 输出 `NO` ✓

---

## 四、关键细节深度解析

### 4.1 为什么 `mid = left + (right - left) / 2`？

```cpp
// 不推荐写法（可能溢出）
int mid = (left + right) / 2;

// 推荐写法（防溢出）
int mid = left + (right - left) / 2;
```

**原因**：当 `left` 和 `right` 都是很大的正整数时，`left + right` 可能超出 `int` 的最大值（约 $2 \times 10^9$），导致**整数溢出**，得到负数或错误结果。

`left + (right - left) / 2` 的数学等价性：

$$left + \frac{right - left}{2} = \frac{2 \cdot left + right - left}{2} = \frac{left + right}{2}$$

两者结果相同，但前者避免了加法溢出。

> 本题数据范围 $|a[i]| \leq 10^9$，`left + right` 最大约 $2 \times 10^5$（下标范围），不会溢出。但这是**良好的编程习惯**，在竞赛和面试中都应使用。

### 4.2 循环条件：为什么是 `left <= right`？

这是**闭区间写法** `[left, right]`，表示搜索范围包含两端点。

| 写法 | 循环条件 | 范围定义 | 边界更新 |
|:----:|:-------:|:-------:|:-------:|
| **闭区间** ⭐ | `left <= right` | `[left, right]` | `right = mid - 1`, `left = mid + 1` |
| 开区间 | `left < right` | `[left, right)` | `right = mid`, `left = mid + 1` |

**闭区间的优势**：
- 逻辑更直观，`left` 和 `right` 都是合法下标；
- 当 `left == right` 时，还有一个元素需要检查；
- 找到目标后直接 `return`，逻辑清晰。

### 4.3 边界更新的正确性

```cpp
if (target < v[mid])  right = mid - 1;   // ✅ 排除 mid 及其右侧
if (target > v[mid])  left = mid + 1;    // ✅ 排除 mid 及其左侧
```

- `right = mid - 1`：既然 `target < v[mid]`，那么 `mid` 位置及其右侧所有元素都 **≥ v[mid] > target**，可以全部排除；
- `left = mid + 1`：既然 `target > v[mid]`，那么 `mid` 位置及其左侧所有元素都 **≤ v[mid] < target**，可以全部排除。

### 4.4 为什么二分查找要求数组有序？

二分查找的核心是"**通过一次比较排除一半元素**"，这依赖于有序性带来的**单调性**：

```
升序数组中：如果 a[mid] < target，则 a[0..mid] 都 < target
            如果 a[mid] > target，则 a[mid..n-1] 都 > target
```

如果数组无序，就无法确定 target 在 mid 的哪一侧，二分查找失效。

---

## 五、复杂度分析

| 维度 | 结果 |
|------|------|
| **单次查询时间** | $O(\log n)$ —— 每次范围减半，最多 $\lceil \log_2 n \rceil$ 次比较 |
| **总查询时间** | $O(m \log n)$ —— $m$ 次查询，每次 $O(\log n)$ |
| **空间复杂度** | $O(n)$ —— 存储数组和查询值（算法本身只用 $O(1)$ 额外空间） |

对于 $n = 10^5, m = 10^5$：
- 总比较次数约 $10^5 \times 17 = 1.7 \times 10^6$，完全可以接受；
- 线性查找需要 $10^{10}$ 次比较，会超时。

---

## 六、易错点与常见错误

### ❌ 错误 1：循环条件写成 `left < right`
```cpp
while (left < right)  // 错误！会漏掉 left == right 的情况
```
当 `left == right` 时，还有一个元素 `v[left]` 没有检查。如果此时 `v[left] == target`，会被漏掉。

### ❌ 错误 2：边界更新写成 `right = mid` 或 `left = mid`
```cpp
// 错误！如果 right = mid，当 left == right - 1 时可能死循环
// 错误！如果 left = mid，当 right == left + 1 时可能死循环
```
闭区间写法必须严格排除 `mid`：`right = mid - 1` 或 `left = mid + 1`。

### ❌ 错误 3：`mid` 计算溢出
```cpp
int mid = (left + right) / 2;  // 不推荐，有溢出风险
```

### ❌ 错误 4：数组未排序就进行二分
二分查找的**前提条件**是数组有序。如果输入无序，结果不可预期。

### ❌ 错误 5：查询值类型与数组元素类型不匹配
```cpp
// 如果数组元素和 target 都是 int，没问题
// 但如果涉及 long long，要确保类型一致
```

---

## 七、扩展与变式

### 7.1 查找第一个/最后一个出现的位置
如果数组中有重复元素，想找到 target 的**第一次**或**最后一次**出现：

```cpp
// 查找第一个 ≥ target 的位置（lower_bound）
while (left < right) {
    int mid = left + (right - left) / 2;
    if (v[mid] < target) left = mid + 1;
    else right = mid;  // v[mid] >= target，不能排除 mid
}

// 查找第一个 > target 的位置（upper_bound）
while (left < right) {
    int mid = left + (right - left) / 2;
    if (v[mid] <= target) left = mid + 1;
    else right = mid;
}
```

### 7.2 使用 STL 的 binary_search
C++ 标准库提供了现成的二分查找函数：

```cpp
#include <algorithm>

// 返回 bool：是否存在
if (binary_search(v.begin(), v.end(), target)) {
    cout << "YES" << endl;
} else {
    cout << "NO" << endl;
}

// 返回迭代器：指向第一个 >= target 的位置
auto it = lower_bound(v.begin(), v.end(), target);
if (it != v.end() && *it == target) {
    cout << "YES" << endl;
} else {
    cout << "NO" << endl;
}
```

> 竞赛中手写二分更灵活，工程中使用 STL 更简洁。

### 7.3 降序数组的二分查找
如果数组是**降序**的，只需调整比较方向：

```cpp
if (target > v[mid]) right = mid - 1;   // 降序：大的在左边
else if (target < v[mid]) left = mid + 1;
```

---

## 八、一句话总结

> **二分查找的本质是"折半排除"：在有序数组中，每次取中间元素比较，根据大小关系果断丢掉一半不可能的区域，$\log n$ 步之内必定找到答案或确认不存在。**

| 要点 | 内容 |
|------|------|
| 前提条件 | 数组必须**有序**（升序或降序） |
| 核心操作 | 取中点 → 比较 → 缩小范围 |
| 中点公式 | `mid = left + (right - left) / 2`（防溢出） |
| 循环条件 | `left <= right`（闭区间写法） |
| 边界更新 | `right = mid - 1` 或 `left = mid + 1` |
| 时间复杂度 | $O(\log n)$ 单次查询 |
| 关键优势 | 比线性查找快数万倍（当 $n = 10^5$ 时） |

---

> 整理日期：2026-07-31  
> 题目来源：P14085.【二分1】查询是否存在某个数
