# 力扣 438. 找到字符串中所有字母异位词 — 学习笔记

## 一、题目概述

给定两个字符串 `s` 和 `p`，找到 `s` 中所有 `p` 的**异位词**的子串，返回这些子串的**起始索引**。

> **异位词（Anagram）**：由相同字符以不同顺序组成的字符串。例如 `"abc"` 和 `"bca"` 是异位词。
>
> 示例：`s = "cbaebabacd"`, `p = "abc"` → 返回 `[0, 6]`（子串 `"cba"` 和 `"bac"` 都是 `"abc"` 的异位词）

---

## 二、滑动窗口 + 哈希表解法

### 核心思想

用**滑动窗口**在 `s` 上维护一个长度等于 `p` 的子串，用**哈希表**比较窗口内的字符频率是否与 `p` 相同。

但直接比较两个哈希表是 $O(26)$ 的，可以优化：
- 维护一个 `matchcount`：**窗口中有多少个字符的种类和数量刚好满足 `p` 的要求**
- 当 `matchcount == targetFreq.size()`（即 `p` 中不同字符的种类数）时，说明窗口是异位词

---

## 三、代码实现（带完整注释）

```cpp
class Solution {
public:
    vector<int> findAnagrams(string s, string p) {
        vector<int> ans;  // 存储所有符合条件的起始索引

        // targetFreq[c] = 字符 c 在 p 中需要出现的次数
        unordered_map<char, int> targetFreq;
        // windowFreq[c] = 字符 c 在当前窗口中实际出现的次数
        unordered_map<char, int> windowFreq;

        // 初始化目标频率表
        for (char i : p) {
            targetFreq[i]++;
        }

        // matchcount = 窗口中有多少种字符的「实际次数」刚好等于「目标次数」
        // 当 matchcount == targetFreq.size() 时，当前窗口就是异位词
        int matchcount = 0;

        int left = 0;   // 窗口左边界
        int right = 0;  // 窗口右边界（左闭右开：[left, right)）

        while (right < s.size()) {
            // ========== 第一步：右边界扩张 ==========
            char curIn = s[right];  // 当前进入窗口的字符

            // 外层判断：这个字符在 p 中出现过吗？
            // 如果 p 中没有这个字符，直接忽略，不影响 matchcount
            if (targetFreq.count(curIn)) {
                windowFreq[curIn]++;

                // 内层判断：窗口中 curIn 的数量刚好达到目标数量吗？
                // 注意：只在「刚好等于」时 matchcount++，超了不加！
                // 详见下方「核心细节」解释
                if (targetFreq[curIn] == windowFreq[curIn]) {
                    matchcount++;
                }
            }
            right++;  // 右指针右移，窗口扩大

            // ========== 第二步：左边界收缩（当窗口大小达到 p.length() 时）==========
            while (right - left == p.size()) {
                // 如果所有字符种类都刚好满足，记录当前左边界
                if (matchcount == targetFreq.size()) {
                    ans.push_back(left);
                }

                char curOut = s[left];  // 当前离开窗口的字符
                left++;  // 左指针右移，窗口缩小

                // 同样只关心 p 中出现的字符
                if (targetFreq.count(curOut)) {
                    // 注意判断顺序：先判断移除前是否刚好满足，再移除！
                    // 如果移除前 curOut 的数量刚好等于目标数量，
                    // 移除后就会从"刚好满足"变成"不足"，matchcount 减 1
                    if (windowFreq[curOut] == targetFreq[curOut]) {
                        matchcount--;
                    }
                    windowFreq[curOut]--;
                }
            }
        }
        return ans;
    }
};
```

---

## 四、变量名对照表

| 变量名 | 含义 | 为什么叫这个名字 |
|--------|------|-----------------|
| `targetFreq` | `p` 中各字符的目标频率 | target frequency，目标频次 |
| `windowFreq` | 当前窗口中各字符的实际频率 | window frequency，窗口频次 |
| `matchcount` | 已匹配的字符种类数 | 有多少种字符刚好满足目标 |
| `left` / `right` | 窗口左右边界 | 滑动窗口的标准命名 |
| `curIn` | 当前进入窗口的字符 | current incoming character |
| `curOut` | 当前离开窗口的字符 | current outgoing character |
| `ans` | 答案数组 | 存储所有符合条件的起始索引 |

---

## 五、核心细节：为什么已经判断了 `count`，还要判断 `==`？

### 两层判断各司其职，完全不重复

| 判断 | 作用 | 问的问题 |
|------|------|---------|
| `if(targetFreq.count(curIn))` | **过滤无关字符** | 这个字符在 `p` 里出现过吗？ |
| `if(targetFreq[curIn] == windowFreq[curIn])` | **判断是否刚好满足** | 这个字符在窗口里的数量**恰好等于**目标数量吗？ |

### 举个例子

`p = "aab"`，所以 `targetFreq['a'] = 2`，`targetFreq['b'] = 1`

窗口滑动时连续遇到 `'a'`：

| 步骤 | 操作 | `windowFreq['a']` | 外层 `count` 成立？ | 内层 `==` 成立？ | `matchcount` | 说明 |
|------|------|-------------------|-------------------|----------------|-------------|------|
| 1 | 遇到第 1 个 `'a'` | 1 | ✅ 是 | ❌ `1 != 2` | 不变 | 有 `'a'`，但数量还不够 |
| 2 | 遇到第 2 个 `'a'` | 2 | ✅ 是 | ✅ `2 == 2` | **+1** | 数量刚好满足！ |
| 3 | 遇到第 3 个 `'a'` | 3 | ✅ 是 | ❌ `3 != 2` | 不变 | 有 `'a'`，但数量超了 |

**如果没有内层判断**，每次外层成立都 `matchcount++`，那第 3 个 `'a'` 会让 `matchcount` 虚高，导致错误地认为窗口已匹配。

### 核心逻辑

`matchcount` 统计的是：**有多少种字符的「实际数量」恰好等于「目标数量」**。

- 少了（`1 < 2`）→ 不算匹配
- 刚好（`2 == 2`）→ 算匹配，`matchcount++`
- 多了（`3 > 2`）→ 不算匹配

只有当**所有字符种类都刚好满足**时，`matchcount == targetFreq.size()` 才成立，此时窗口才是异位词。

> 所以外层是「有没有」，内层是「够不够且不多」，完全是两回事。

---

## 六、另一个关键细节：左边界收缩时的判断顺序

```cpp
if (windowFreq[curOut] == targetFreq[curOut]) {
    matchcount--;
}
windowFreq[curOut]--;
```

**注意：`matchcount--` 是在 `windowFreq--` 之前判断的！**

为什么？
- 假设 `targetFreq['a'] = 2`，当前 `windowFreq['a'] = 2`
- 移除前 `windowFreq['a'] == targetFreq['a']` 成立，说明移除后就不满足了
- 所以先 `matchcount--`，再 `windowFreq--`

> 如果顺序反过来，判断时 `windowFreq` 已经变了，逻辑就错了。

---

## 七、流程图解（示例 `s = "cbaebabacd"`, `p = "abc"`）

```
p = "abc" → targetFreq: {a:1, b:1, c:1}, 需要 matchcount == 3

初始: left=0, right=0, matchcount=0

right=0, curIn='c': window['c']=1, matchcount=0 (1≠1)
right=1, 窗口大小=1 < 3, 不收缩

right=1, curIn='b': window['b']=1, matchcount=0 (1≠1)
right=2, 窗口大小=2 < 3, 不收缩

right=2, curIn='a': window['a']=1, matchcount=1 (1==1)
         window['b']=1 也满足 → matchcount=2
         window['c']=1 也满足 → matchcount=3 ✅
right=3, 窗口大小=3 == 3, 收缩:
         matchcount(3) == targetFreq.size()(3) → ans=[0]
         curOut='c': window['c']==1==target → matchcount=2, window['c']=0
         left=1

right=3, curIn='e': 'e' 不在 targetFreq 中，忽略
right=4, 窗口大小=3 == 3, 收缩:
         matchcount(2) ≠ 3 → 不记录
         curOut='b': window['b']==1==target → matchcount=1, window['b']=0
         left=2

...（中间步骤省略）...

right=8, curIn='a': window['a']=1, matchcount 逐步恢复
right=9, 窗口大小=3 == 3, 收缩:
         matchcount(3) == 3 → ans=[0, 6]
         ...

最终结果: [0, 6] ✅
```

---

## 八、进一步优化：用数组代替哈希表

由于题目说明字符串只包含**小写英文字母**，可以用固定大小的数组代替 `unordered_map`，效率更高：

```cpp
class Solution {
public:
    vector<int> findAnagrams(string s, string p) {
        vector<int> ans;
        if (s.size() < p.size()) return ans;

        // 用数组记录频率，下标 0~25 对应 'a'~'z'
        int targetFreq[26] = {0};
        int windowFreq[26] = {0};

        for (char c : p) {
            targetFreq[c - 'a']++;
        }

        // 统计 p 中有多少种不同的字符
        int targetTypes = 0;
        for (int i = 0; i < 26; i++) {
            if (targetFreq[i] > 0) targetTypes++;
        }

        int matchcount = 0;
        int left = 0, right = 0;

        while (right < s.size()) {
            char curIn = s[right];
            int idx = curIn - 'a';

            if (targetFreq[idx] > 0) {
                windowFreq[idx]++;
                if (windowFreq[idx] == targetFreq[idx]) {
                    matchcount++;
                }
            }
            right++;

            while (right - left == p.size()) {
                if (matchcount == targetTypes) {
                    ans.push_back(left);
                }

                char curOut = s[left];
                int outIdx = curOut - 'a';
                left++;

                if (targetFreq[outIdx] > 0) {
                    if (windowFreq[outIdx] == targetFreq[outIdx]) {
                        matchcount--;
                    }
                    windowFreq[outIdx]--;
                }
            }
        }

        return ans;
    }
};
```

| 对比 | `unordered_map` 版 | 数组版 |
|------|-------------------|--------|
| 时间 | $O(n)$（但有哈希开销） | $O(n)$（数组访问 $O(1)$ 更快） |
| 空间 | $O(k)$（$k$ 为不同字符数） | $O(1)$（固定 26 个整数） |
| 适用 | 通用字符集 | 仅小写字母 |

---

## 九、复杂度分析

| 指标 | 复杂度 | 说明 |
|------|--------|------|
| **时间复杂度** | $O(n)$ | 左右指针各遍历一次字符串，`n = s.length()` |
| **空间复杂度** | $O(k)$ / $O(1)$ | 哈希表版 $O(k)$（$k$ 为 `p` 中不同字符数），数组版 $O(1)$ |
| **是否修改原字符串** | ❌ 否 | 只读遍历 |

---

## 十、关键收获

1. **滑动窗口是处理「固定长度子串匹配」问题的利器**。窗口大小始终等于 `p.length()`，右扩一步、左缩一步，保持窗口长度不变。
2. **`matchcount` 的维护是精髓**：
   - 外层 `count`：过滤无关字符（`p` 中没有的字符直接忽略）
   - 内层 `==`：只在「刚好等于目标次数」时 `++`，超了不加，少了不减
   - 两层判断各司其职，缺一不可
3. **判断顺序很重要**：左边界收缩时，`matchcount--` 必须在 `windowFreq--` **之前**执行，因为判断依赖的是移除前的状态。
4. **数组优化**：当字符集有限（如 26 个小写字母）时，用固定数组代替哈希表，既省空间又省时间。
5. **滑动窗口的通用模板**：
   ```cpp
   int left = 0, right = 0;
   while (right < n) {
       // 右边界扩张，加入 s[right]
       // ...
       right++;

       // 窗口满足条件时，收缩左边界
       while (窗口需要收缩) {
           // 记录答案
           // 左边界收缩，移除 s[left]
           // ...
           left++;
       }
   }
   ```
