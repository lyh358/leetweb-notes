# LeetCode 42 接雨水 学习笔记（双指针解法）

> 题目链接：https://leetcode.cn/problems/trapping-rain-water/
> 难度：困难
> 标签：数组、双指针、动态规划、单调栈

---

## 1. 题目分析

### 题目描述
给定 `n` 个非负整数表示每个宽度为 1 的柱子的高度图，计算按此排列的柱子，下雨之后能接多少雨水。

### 示例

```
输入：height = [0,1,0,2,1,0,1,3,2,1,2,1]
输出：6
解释：上面是由数组 [0,1,0,2,1,0,1,3,2,1,2,1] 表示的高度图，在这种情况下，可以接 6 个单位的雨水。
```

```
输入：height = [4,2,0,3,2,5]
输出：9
```

### 核心问题
每个位置能接多少水？

**答案：取决于它左边最高的柱子和右边最高的柱子中，较矮的那一个。**

> 木桶原理：一个桶能装多少水，取决于最短的那块木板。
> 接雨水：一个位置能接多少水，取决于左右两边最高柱子中较矮的那根，减去当前柱子高度。

```
water[i] = min(left_max[i], right_max[i]) - height[i]
```

---

## 2. 思路演变

### 暴力解法（超时）
对每个位置，分别往左找最大值、往右找最大值，计算接水量。
- 时间 O(n²)，空间 O(1)
- n 最大 2×10⁴，4×10⁸ 次操作，超时。

### 动态规划（预处理）
提前用两个数组存每个位置的左边最大值、右边最大值。
- 时间 O(n)，空间 O(n)
- 需要两个额外数组

### 双指针（最优）
用左右指针从两端向中间移动，边走边维护左右最大值。
- 时间 O(n)，空间 O(1)
- 空间最优，也是面试常考写法

### 单调栈
按行计算接水量，用单调递减栈。
- 时间 O(n)，空间 O(n)
- 思路不同，适合拓展学习

> 本文档重点讲解**双指针解法**。

---

## 3. 双指针解法

### 核心思想
用两个指针从两端向中间移动，同时维护左边最大值 `left_max` 和右边最大值 `right_max`。

**关键结论**：
- 如果 `left_max < right_max`，说明**左边是短板**。左指针位置的接水量由 `left_max` 决定（因为右边一定有比 left_max 更高的柱子）。
- 如果 `left_max >= right_max`，说明**右边是短板**。右指针位置的接水量由 `right_max` 决定。

### 为什么这个结论是对的？

想象一下：
- left 指针从左往右走，left_max 是 left 左边（含left）的最高柱子。
- right 指针从右往左走，right_max 是 right 右边（含right）的最高柱子。

如果 `left_max < right_max`：
- left 位置右边一定有一根柱子 >= right_max > left_max
- 所以 left 位置的"右边最高"肯定 > left_max
- 那么 min(左边最高, 右边最高) = left_max
- left 位置接水量 = left_max - height[left]

反之同理。

---

## 4. 双指针代码

```cpp
#include <vector>
#include <algorithm>
using namespace std;

class Solution {
public:
    int trap(vector<int>& height) {
        int n = height.size();
        if (n <= 2) return 0;   // 至少3根柱子才能接水
        
        int left = 0;
        int right = n - 1;
        int left_max = 0;
        int right_max = 0;
        int ans = 0;
        
        while (left < right) {
            if (height[left] < height[right]) {
                // 左边矮，左边是短板，处理左指针
                if (height[left] >= left_max) {
                    left_max = height[left];   // 更新左边最高
                } else {
                    ans += left_max - height[left];   // 接水
                }
                left++;
            } else {
                // 右边矮，右边是短板，处理右指针
                if (height[right] >= right_max) {
                    right_max = height[right];   // 更新右边最高
                } else {
                    ans += right_max - height[right];   // 接水
                }
                right--;
            }
        }
        return ans;
    }
};
```

---

## 5. 代码逐行解析

### 5.1 初始化
```cpp
int left = 0;          // 左指针，从最左端开始
int right = n - 1;     // 右指针，从最右端开始
int left_max = 0;      // 左边最高柱子高度
int right_max = 0;     // 右边最高柱子高度
int ans = 0;           // 接水总量
```

### 5.2 主循环
```cpp
while (left < right) {
    if (height[left] < height[right]) {
        // 处理左指针...
        left++;
    } else {
        // 处理右指针...
        right--;
    }
}
```

**移动规则**：哪边柱子矮，就移动哪边的指针。
- 左边矮 → 左边是短板 → 左指针位置的接水量可以确定 → left++
- 右边矮 → 右边是短板 → 右指针位置的接水量可以确定 → right--

### 5.3 处理左指针
```cpp
if (height[left] >= left_max) {
    left_max = height[left];   // 当前柱子比左边最高还高，更新最大值
} else {
    ans += left_max - height[left];   // 否则可以接水，水量 = 最高 - 当前高度
}
left++;
```

两种情况：
1. **当前柱子 >= left_max**：这根柱子更高，更新 left_max，接不了水（它本身就是边界）。
2. **当前柱子 < left_max**：左边有更高的柱子挡着，右边也一定有更高的柱子（因为 height[left] < height[right]，且 right_max >= height[right]），所以能接水。

### 5.4 处理右指针
```cpp
if (height[right] >= right_max) {
    right_max = height[right];
} else {
    ans += right_max - height[right];
}
right--;
```
和左指针对称。

---

## 6. 手动模拟一遍

以 `height = [0,1,0,2,1,0,1,3,2,1,2,1]` 为例：

```
初始：left=0, right=11, left_max=0, right_max=0, ans=0

height[0]=0 < height[11]=1 → 处理左
  height[0]=0 >= left_max=0 → left_max=0
  left=1

height[1]=1 > height[11]=1 → 处理右（相等也走右）
  height[11]=1 >= right_max=0 → right_max=1
  right=10

height[1]=1 < height[10]=2 → 处理左
  height[1]=1 >= left_max=0 → left_max=1
  left=2

height[2]=0 < height[10]=2 → 处理左
  height[2]=0 < left_max=1 → ans += 1-0 = 1
  left=3

height[3]=2 < height[10]=2 → 处理左（相等走左）
  height[3]=2 >= left_max=1 → left_max=2
  left=4

height[4]=1 < height[10]=2 → 处理左
  height[4]=1 < left_max=2 → ans += 2-1 = 2
  left=5

height[5]=0 < height[10]=2 → 处理左
  height[5]=0 < left_max=2 → ans += 2-0 = 4
  left=6

height[6]=1 < height[10]=2 → 处理左
  height[6]=1 < left_max=2 → ans += 2-1 = 5
  left=7

height[7]=3 > height[10]=2 → 处理右
  height[10]=2 >= right_max=1 → right_max=2
  right=9

height[7]=3 > height[9]=1 → 处理右
  height[9]=1 < right_max=2 → ans += 2-1 = 6
  right=8

height[7]=3 > height[8]=2 → 处理右
  height[8]=2 < right_max=2 → 相等，不加
  right=7

left=7, right=7 → 循环结束

ans = 6 ✓
```

---

## 7. 涉及知识点

### 7.1 对撞双指针
- 左右指针从两端向中间移动
- 每一步都能确定一个位置的答案
- 把 O(n²) 降到 O(n)

### 7.2 木桶原理（短板效应）
- 接水量由较短的那一侧决定
- 双指针的移动规则本质就是：哪边是短板，就处理哪边

### 7.3 空间优化思想
- 动态规划需要两个数组存 left_max 和 right_max
- 双指针只用两个变量，边走边算，空间从 O(n) 降到 O(1)

---

## 8. 时间、空间复杂度

| 复杂度 | 分析 |
|--------|------|
| **时间 O(n)** | 左右指针从两端向中间移动，每个位置最多访问一次 |
| **空间 O(1)** | 只用了几个变量，没有额外数据结构 |

> 双指针是接雨水的最优解法（时间空间都是最优）。

---

## 9. 常见解法对比

| 解法 | 时间 | 空间 | 思路 |
|------|------|------|------|
| 暴力 | O(n²) | O(1) | 每个位置左右找最大值（超时） |
| 动态规划 | O(n) | O(n) | 预处理 left_max、right_max 数组 |
| **双指针** | **O(n)** | **O(1)** | 边走边维护左右最大值（最优） |
| 单调栈 | O(n) | O(n) | 按行计算，单调递减栈 |

---

## 10. 易错点总结

1. **移动规则搞反**：哪边矮移动哪边，不是哪边高移动哪边。
2. **先更新 max 还是先算水**：先判断当前高度和 max 的关系，如果更高就更新 max（接不了水），否则才算接水量。
3. **相等时怎么办**：height[left] == height[right] 时，移动哪边都可以，不影响结果。
4. **left_max 初始值**：初始为 0 就行，第一根柱子肯定接不了水。
5. **边界条件**：柱子数 <= 2 时接不了水，直接返回 0。
6. **和"盛最多水的容器"搞混**：
   - 盛最多水：选两根柱子，面积 = 宽 × 较短高度，找最大面积
   - 接雨水：所有柱子都用上，计算总共能接多少水
   - 两道题完全不同，解法也不同

---

## 11. 接雨水 vs 盛最多水的容器

| 对比项 | 盛最多水的容器（LeetCode 11） | 接雨水（LeetCode 42） |
|--------|------------------------------|----------------------|
| 问题 | 选两根柱子，最大面积是多少 | 所有柱子之间，总共接多少水 |
| 计算单位 | 一个容器的面积 | 每个位置的接水量，累加 |
| 双指针移动规则 | 移动较矮的指针（寻找更大面积） | 移动较矮的指针（确定当前位置接水量） |
| 时间复杂度 | O(n) | O(n) |
| 空间复杂度 | O(1) | O(1) |

> 两道题双指针长得很像，但问题本质完全不同，别搞混了！

---

## 12. 拓展：单调栈解法思路

双指针是"按列"计算（每个位置能接多少水），单调栈是"按行"计算（每一层能接多少水）。

简单思路：
- 维护一个单调递减栈（栈底到栈顶高度递减）
- 遇到比栈顶高的柱子，说明形成了凹槽，可以接水
- 弹出栈顶，计算这一行的接水量
- 宽度 = 当前索引 - 新栈顶索引 - 1
- 高度 = min(当前高度, 新栈顶高度) - 弹出的高度

有兴趣可以单独研究，面试也可能考。******粗体******