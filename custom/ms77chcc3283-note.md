# LeetCode 18 四数之和 学习笔记

> 题目链接：https://leetcode.cn/problems/4sum/
> 难度：中等
> 标签：数组、双指针、排序、去重

---

## 1. 题目分析

### 题目描述
给你一个由 `n` 个整数组成的数组 `nums`，和一个目标值 `target`。请你找出并返回满足下述全部条件且**不重复**的四元组 `[nums[a], nums[b], nums[c], nums[d]]`（若两个四元组元素一一对应，则认为两个四元组重复）：

- `0 <= a, b, c, d < n`
- `a、b、c 和 d 互不相同`
- `nums[a] + nums[b] + nums[c] + nums[d] == target`

你可以按**任意顺序**返回答案。

### 示例
```
输入：nums = [1,0,-1,0,-2,2], target = 0
输出：[[-2,-1,1,2],[-2,0,0,2],[-1,0,0,1]]
```

```
输入：nums = [2,2,2,2,2], target = 8
输出：[[2,2,2,2]]
```

### 核心难点
1. **去重**：四个数都要去重，比三数之和更复杂。
2. **溢出**：四个 int 相加可能溢出，需要用 long long。
3. **剪枝优化**：层数多，合理剪枝能大幅提速。

---

## 2. 思路梳理

### 整体思路：再降一层
三数之和是"固定1个 + 双指针找2个"，四数之和就是：
- **固定2个数**（两层循环）+ **双指针找后2个**
- 本质就是在三数之和外面再套一层循环

### 步骤
1. 排序数组。
2. 第一层循环：固定第一个数 `nums[i]`，去重。
3. 第二层循环：固定第二个数 `nums[j]`（j = i + 1），去重。
4. 双指针 `left = j + 1`，`right = n - 1`，找后两个数。
5. 每层都要去重，同时可以做剪枝优化。

---

## 3. 正确代码

```cpp
#include <vector>
#include <algorithm>
using namespace std;

class Solution {
public:
    vector<vector<int>> fourSum(vector<int>& nums, int target) {
        vector<vector<int>> ans;
        int n = nums.size();
        if (n < 4) return ans;
        
        sort(nums.begin(), nums.end());
        
        for (int i = 0; i < n - 3; i++) {
            // 去重：第一个数
            if (i > 0 && nums[i] == nums[i - 1]) continue;
            
            // 剪枝：最小的四个数都大于target，后面不可能了
            if ((long long)nums[i] + nums[i + 1] + nums[i + 2] + nums[i + 3] > target) break;
            // 剪枝：当前数加最大的三个数都小于target，这个数不行，换下一个
            if ((long long)nums[i] + nums[n - 1] + nums[n - 2] + nums[n - 3] < target) continue;
            
            for (int j = i + 1; j < n - 2; j++) {
                // 去重：第二个数
                if (j > i + 1 && nums[j] == nums[j - 1]) continue;
                
                // 剪枝：最小的四个数都大于target
                if ((long long)nums[i] + nums[j] + nums[j + 1] + nums[j + 2] > target) break;
                // 剪枝：当前两个数加最大的两个数都小于target
                if ((long long)nums[i] + nums[j] + nums[n - 1] + nums[n - 2] < target) continue;
                
                int left = j + 1;
                int right = n - 1;
                
                while (left < right) {
                    long long sum = (long long)nums[i] + nums[j] + nums[left] + nums[right];
                    if (sum == target) {
                        ans.push_back({nums[i], nums[j], nums[left], nums[right]});
                        // 去重：左指针
                        while (left < right && nums[left] == nums[left + 1]) left++;
                        // 去重：右指针
                        while (left < right && nums[right] == nums[right - 1]) right--;
                        left++;
                        right--;
                    } else if (sum < target) {
                        left++;
                    } else {
                        right--;
                    }
                }
            }
        }
        return ans;
    }
};
```

---

## 4. 代码逐段解析

### 4.1 排序 + 边界
```cpp
sort(nums.begin(), nums.end());
if (n < 4) return ans;
```
和三数之和一样，排序是基础。

### 4.2 第一层循环 i（第一个数）
```cpp
for (int i = 0; i < n - 3; i++) {
    if (i > 0 && nums[i] == nums[i - 1]) continue;  // 去重
    
    // 剪枝1：最小四个数都 > target，直接break
    if ((long long)nums[i] + nums[i+1] + nums[i+2] + nums[i+3] > target) break;
    // 剪枝2：当前数 + 最大三个数 < target，这个i不行，换下一个
    if ((long long)nums[i] + nums[n-1] + nums[n-2] + nums[n-3] < target) continue;
    // ...
}
```

**循环条件 `i < n - 3`**：后面至少要留3个数（j、left、right），所以 i 最多到 n-4。

**两个剪枝**：
- **break 剪枝**：最小的四个数（连续四个）都大于 target，后面更大，直接跳出整个循环。
- **continue 剪枝**：当前数加上最大的三个数都小于 target，说明这个 i 太小了，跳过这个 i，试下一个。

### 4.3 第二层循环 j（第二个数）
```cpp
for (int j = i + 1; j < n - 2; j++) {
    if (j > i + 1 && nums[j] == nums[j - 1]) continue;  // 去重
    
    // 同样的剪枝逻辑
    if ((long long)nums[i] + nums[j] + nums[j+1] + nums[j+2] > target) break;
    if ((long long)nums[i] + nums[j] + nums[n-1] + nums[n-2] < target) continue;
    // ...
}
```

**j 的去重条件 `j > i + 1`**：
- j 从 i+1 开始，所以 j 的去重要和 j-1 比，但前提是 j > i+1（不是第一个 j）。
- 注意是 `j > i + 1`，不是 `j > 0`，因为 j 的起点是 i+1 不是 0。

### 4.4 双指针找后两个数
```cpp
int left = j + 1;
int right = n - 1;

while (left < right) {
    long long sum = (long long)nums[i] + nums[j] + nums[left] + nums[right];
    if (sum == target) {
        ans.push_back({nums[i], nums[j], nums[left], nums[right]});
        while (left < right && nums[left] == nums[left + 1]) left++;   // 去重
        while (left < right && nums[right] == nums[right - 1]) right--; // 去重
        left++;
        right--;
    } else if (sum < target) {
        left++;
    } else {
        right--;
    }
}
```
和三数之和的双指针逻辑完全一样。

---

## 5. 重点：溢出问题（本题大坑）

### 为什么会溢出？
四个 int 相加，每个 int 最大约 2×10⁹，四个加起来最大约 8×10⁹，而 int 的范围约是 -2×10⁹ ~ 2×10⁹，**四个 int 相加会超出 int 范围，导致整数溢出**。

### 解决方法：强制转 long long
```cpp
long long sum = (long long)nums[i] + nums[j] + nums[left] + nums[right];
```
- 把第一个数强转成 `long long`，后面的数会自动提升为 long long。
- 整个表达式就用 long long 计算，不会溢出。

> ⚠️ 不要写成 `long long sum = nums[i] + nums[j] + nums[left] + nums[right];`
> 这样是先按 int 相加（已经溢出了），再赋值给 long long，已经晚了。

### 剪枝里也要注意溢出
所有四个数相加的地方，都要用 long long：
```cpp
if ((long long)nums[i] + nums[i+1] + nums[i+2] + nums[i+3] > target) break;
```

---

## 6. 四重去重总结

| 位置 | 写法 | 注意点 |
|------|------|--------|
| 第一个数 i | `if (i>0 && nums[i]==nums[i-1]) continue;` | 和前一个比 |
| 第二个数 j | `if (j>i+1 && nums[j]==nums[j-1]) continue;` | 注意是 `j > i+1` 不是 `j > 0` |
| 左指针 left | `while (left<right && nums[left]==nums[left+1]) left++;` | while 一次性跳完 |
| 右指针 right | `while (left<right && nums[right]==nums[right-1]) right--;` | while 一次性跳完 |

---

## 7. 剪枝优化详解

### 第一层 i 的剪枝
```cpp
// 剪枝1：break型，当前i开头最小的四数都 > target，后面更大，直接结束
if (nums[i] + nums[i+1] + nums[i+2] + nums[i+3] > target) break;

// 剪枝2：continue型，当前i搭配最大三个数都 < target，这个i太小了，跳过
if (nums[i] + nums[n-1] + nums[n-2] + nums[n-3] < target) continue;
```

### 第二层 j 的剪枝
```cpp
// 同理
if (nums[i] + nums[j] + nums[j+1] + nums[j+2] > target) break;
if (nums[i] + nums[j] + nums[n-1] + nums[n-2] < target) continue;
```

### 剪枝效果
不加剪枝也能过，但加了剪枝运行时间能从几百毫秒降到十几毫秒，优化很明显。

---

## 8. 时间、空间复杂度

设 n 为数组长度。

| 复杂度 | 分析 |
|--------|------|
| **时间 O(n³)** | 两层循环 × 双指针 = n × n × n = O(n³) |
| **空间 O(log n) ~ O(n)** | 排序的空间开销，结果数组不计入 |

> 对比：三数之和 O(n²)，四数之和 O(n³)，k 数之和 O(n^(k-1))。

---

## 9. 易错点总结

1. **溢出！溢出！溢出！** 四个 int 相加必须用 long long，强转第一个数。
2. **j 的去重条件写错**：应该是 `j > i + 1`，不是 `j > 0`，也不是 `j > 1`。
3. **忘记剪枝**：不剪枝也能过，但会慢很多。
4. **循环范围写错**：i 要 `< n - 3`，j 要 `< n - 2`，后面要留够位置。
5. **去重时机**：找到解之后再去重，不要提前去重，否则可能漏掉解。
6. **target 可以是负数**：不要假设 target 是 0 或正数，剪枝判断要小心。

---

## 10. 三数之和 vs 四数之和 对比

| 对比项 | 三数之和 | 四数之和 |
|--------|---------|---------|
| 固定几个数 | 1个（i） | 2个（i, j） |
| 时间复杂度 | O(n²) | O(n³) |
| 溢出风险 | 一般不会 | 必须用 long long |
| 去重层数 | 3层（i + 双指针） | 4层（i + j + 双指针） |
| 剪枝 | 可以加 | 更有必要加 |
| 核心思想 | 排序 + 双指针 | 排序 + 双指针（再套一层） |

---

## 11. 推广：k 数之和通用模板

所有 k 数之和都可以用同一个思路：
1. 排序
2. 前 k-2 个数用循环固定
3. 最后两个数用双指针
4. 每层都去重 + 剪枝

时间复杂度统一是 O(n^(k-1))。

> 