# 力扣 3. 无重复字符的最长子串 — 学习笔记

## 一、题目概述

给定一个字符串 `s`，请你找出其中**不含有重复字符的「最长子串」**的长度。

> 子串：连续的子序列，不是子序列。
>
> 示例：`"abcabcbb"` → 最长无重复子串是 `"abc"`，长度为 `3`。

---

## 二、滑动窗口解法（最优）

### 核心思想

想象一个**窗口**在字符串上从左向右滑动：
- 窗口的**左边界** `left`：当前无重复子串的起始位置
- 窗口的**右边界** `i`：当前遍历到的字符位置
- 遇到重复字符时，**收缩左边界**，把重复字符踢出窗口
- 每次右边界前进时，更新最大长度

### 为什么用「滑动窗口」？

因为子串必须是**连续**的。当我们从左到右遍历字符串时，如果发现当前字符和窗口内的某个字符重复了，我们只需要把左边界移到那个重复字符的**下一个位置**，而不是从头开始。这样右边界始终只走一遍，时间复杂度就是 $O(n)$。

---

## 三、代码实现（带完整注释）

```cpp
class Solution {
public:
    int lengthOfLongestSubstring(string s) {
        // lastPos[curChar] 记录字符 c 上次出现位置的「下一个位置」
        // 即：如果字符 c 在索引 i 处出现，lastPos[curChar] = i + 1
        // 这样当 c 再次出现时，可以直接把左边界跳到 i + 1
        // 数组大小 128 覆盖所有 ASCII 字符
        vector<int> lastPos(128, 0);

        int left = 0;      // 当前无重复子串的左边界（窗口起点）
        int maxLen = 0;    // 记录最长子串的长度

        for (int i = 0; i < s.size(); i++) {
            // 当前字符 s[i]
            char curChar = s[i];

            // 如果字符 c 之前出现过，且它的上次位置在窗口内（>= left）
            // 就把左边界 left 移到它上次出现位置的下一个位置
            // 用 max 保证 left 不会回退（比如 "abba" 的情况）
            left = max(left, lastPos[curChar]);

            // 更新最大长度：当前窗口大小 = i - left + 1
            maxLen = max(maxLen, i - left + 1);

            // 记录字符 c 的「下次可以作为 left 的位置」
            // 为什么是 i + 1？因为下次遇到重复时，left 应该跳到 i 的下一个位置
            lastPos[curChar] = i + 1;
        }

        return maxLen;
    }
};
```

---

## 四、变量名说明

| 变量名 | 含义 | 为什么叫这个名字 |
|--------|------|-----------------|
| `lastPos` | 字符上次出现位置的「下一个位置」 | last position + 1，即下次可以作为窗口起点的位置 |
| `left` | 窗口左边界 | 当前无重复子串的最左索引 |
| `maxLen` | 最大长度 | maximum length 的缩写 |
| `i` | 窗口右边界 | 当前遍历到的字符索引 |

> 💡 为什么不叫 `map`？因为 `map` 容易和 C++ STL 的 `std::map` 混淆，用 `lastPos` 更直观。

---

## 五、关键细节：为什么 `lastPos` 存的是 `i + 1`？

假设字符串 `"abca"`，遍历到第二个 `'a'`（索引 3）时：

| 步骤 | i | 字符 | lastPos['a'] | left | 窗口 | maxLen | 说明 |
|------|---|------|---------------|------|------|--------|------|
| 0 | 0 | 'a' | 0 | 0 | "a" | 1 | lastPos['a'] = 1 |
| 1 | 1 | 'b' | 0 | 0 | "ab" | 2 | lastPos['b'] = 2 |
| 2 | 2 | 'c' | 0 | 0 | "abc" | 3 | lastPos['c'] = 3 |
| 3 | 3 | 'a' | **1** | **1** | "bca" | 3 | left = max(0, 1) = 1，lastPos['a'] = 4 |

当遇到第二个 `'a'` 时：
- `lastPos['a']` 存的是 `1`（第一个 `'a'` 的下一个位置）
- `left = max(0, 1) = 1`，窗口从 `"abc"` 收缩为 `"bca"`
- 新的 `lastPos['a']` 更新为 `4`（当前索引 3 + 1）

如果存的是 `i` 而不是 `i + 1`，那 `left` 就要写成 `lastPos[curChar] + 1`，代码会多一步。存 `i + 1` 是为了让 `left` 的更新更直接。

---

## 六、为什么 `left = max(left, lastPos[curChar])` 不能省略 `max`？

考虑字符串 `"abba"`：

| 步骤 | i | 字符 | lastPos | left（不加 max） | left（加 max） | 说明 |
|------|---|------|---------|------------------|----------------|------|
| 0 | 0 | 'a' | a=1 | 0 | 0 | — |
| 1 | 1 | 'b' | b=2 | 0 | 0 | — |
| 2 | 2 | 'b' | b=3 | 2 | 2 | left = max(0, 2) = 2，窗口 "ba" |
| 3 | 3 | 'a' | a=4 | **1** | **2** | 如果不加 max，left 会回退到 1！|

在第 3 步：
- `lastPos['a'] = 1`（第一个 'a' 的位置 + 1）
- 但此时 `left` 已经是 `2` 了（因为第 2 步遇到 'b' 重复，left 跳到了 2）
- 如果不加 `max`，`left` 会回退到 `1`，导致窗口 `"bba"` 里还有重复的 'b'
- 加 `max` 后，`left = max(2, 1) = 2`，保持不回退，窗口正确为 `"ba"`

> ⚠️ **`max` 是保证 `left` 只向右移动、不回退的关键！**

---

## 七、流程图解（示例 `"abcabcbb"`）

```
初始: left=0, maxLen=0

i=0, 'a': lastPos['a']=0（下次遇到 'a' 时 left 跳转到 0） → left=max(0,0)=0 → maxLen=max(0,1)=1 → lastPos['a']=1
     窗口: [a]bcabcbb

i=1, 'b': lastPos['b']=0（下次遇到 'b' 时 left 跳转到 0） → left=max(0,0)=0 → maxLen=max(1,2)=2 → lastPos['b']=2
     窗口: [ab]cabcbb

i=2, 'c': lastPos['c']=0（下次遇到 'c' 时 left 跳转到 0） → left=max(0,0)=0 → maxLen=max(2,3)=3 → lastPos['c']=3
     窗口: [abc]abcbb

i=3, 'a': lastPos['a']=1 → left=max(0,1)=1 → maxLen=max(3,3)=3 → lastPos['a']=4
     窗口: a[bca]bcbb

i=4, 'b': lastPos['b']=2 → left=max(1,2)=2 → maxLen=max(3,3)=3 → lastPos['b']=5
     窗口: ab[cab]cbb

i=5, 'c': lastPos['c']=3 → left=max(2,3)=3 → maxLen=max(3,3)=3 → lastPos['c']=6
     窗口: abc[abc]bb

i=6, 'b': lastPos['b']=5 → left=max(3,5)=5 → maxLen=max(3,2)=3 → lastPos['b']=7
     窗口: abca[bc]b

i=7, 'b': lastPos['b']=7 → left=max(5,7)=7 → maxLen=max(3,1)=3 → lastPos['b']=8
     窗口: abcabcb[b]

结果: maxLen = 3 ✅
```

---

## 八、复杂度分析

| 指标 | 复杂度 | 说明 |
|------|--------|------|
| **时间复杂度** | $O(n)$ | 右边界 `i` 只遍历一次，每个字符处理一次 |
| **空间复杂度** | $O(1)$ | `lastPos` 数组大小固定为 128（ASCII 字符集） |
| **是否修改原字符串** | ❌ 否 | 只读遍历 |

---

## 九、关键收获

1. **滑动窗口是处理「连续子串/子数组」问题的经典套路**。维护一个左边界和一个右边界，右边界前进，左边界按需收缩。
2. **`lastPos` 存 `i + 1` 而不是 `i`**，是为了让左边界更新更直接，少写一次 `+1`。
3. **`max(left, lastPos[curChar])` 是保证正确性的核心**。不加 `max` 会导致左边界回退，产生错误结果（如 `"abba"` 会算出 3 而不是 2）。
4. **数组大小 128 是 ASCII 字符集**。如果字符串包含 Unicode，需要改用 `unordered_map`。
5. **滑动窗口的通用模板**：
   ```cpp
   int left = 0;
   for (int i = 0; i < n; i++) {
       // 扩展右边界，处理 s[i]
       // 收缩左边界，直到窗口满足条件
       left = max(left, ...);
       // 更新答案
       ans = max(ans, i - left + 1);
   }
   ```
