# 📱 回溯算法学习笔记 —— 力扣 17. 电话号码的字母组合（C++）

> 重点解答两个核心疑问：
> 1. 为什么 `for` 循环从 `i=0` 开始，而不是 `start`？
> 2. 为什么 `unordered_map` 必须定义在类级别，不能放在函数里？

---

## 📌 一、题目：LeetCode 17. 电话号码的字母组合

**题目链接**：[17. 电话号码的字母组合](https://leetcode.cn/problems/letter-combinations-of-a-phone-number/)

**题目描述**：给定一个仅包含数字 `2-9` 的字符串，返回所有它能表示的字母组合。答案可以按**任意顺序**返回。给出数字到字母的映射如下（与电话按键相同）。注意 `1` 不对应任何字母。

**数字-字母映射**：

```
2: abc    3: def
4: ghi    5: jkl    6: mno
7: pqrs   8: tuv    9: wxyz
```

**示例**：

```
输入: digits = "23"
输出: ["ad", "ae", "af", "bd", "be", "bf", "cd", "ce", "cf"]
```

**约束**：
- `0 <= digits.length <= 4`
- `digits[i]` 是范围 `['2', '9']` 的一个数字

---

## 🧠 二、核心思路：多层选择的笛卡尔积

这道题的本质是：**对于输入字符串的每一位数字，从它对应的字母中选择一个，所有选择的笛卡尔积就是答案。**

比如 `digits = "23"`：
- 第1位 `2` 对应 `"abc"`，有 3 种选择
- 第2位 `3` 对应 `"def"`，有 3 种选择
- 总组合数 = 3 × 3 = 9 种

### 递归参数 `start` 的含义

在这道题中，`start` **不是**「从哪个下标开始选字母」，而是**「当前处理到输入字符串的第几个数字」**。

```cpp
void backtracing(..., int start) {
    if (start == digits.length()) {  // 所有数字都处理完了
        combinations.push_back(combination);
        return;
    }

    string digit = phoneMap[digits[start]];  // 取出当前数字对应的字母串
    // 遍历这个字母串的每一个字母
    for (int i = 0; i < digit.length(); i++) {  // ← 这里从0开始！
        ...
    }
}
```

---

## ❓ 三、问题一：为什么 `for` 从 `i=0` 开始，而不是 `start`？

### 3.1 先回顾组合问题中为什么用 `start`

在组合问题（LeetCode 77）中：
- 候选集是一个**固定的数字池** `[1, 2, ..., n]`
- 我们要从中**无序地**选出 `k` 个
- 为了避免 `[1,2]` 和 `[2,1]` 重复，必须规定「只能往后选」
- 所以用 `start` 记录「上一层的数是什么」，下一层从 `start+1` 开始

```cpp
// 组合问题：候选集是 [1..n]，要防重复
for (int i = start; i <= n; i++) {  // ← 从 start 开始
    pack.push_back(i);
    backtrack(n, k, i + 1);         // 下一层只能选更大的
    pack.pop_back();
}
```

### 3.2 电话号码问题为什么从 0 开始

在电话号码问题中：
- **每一层的候选集完全不同**！第1层是 `"abc"`，第2层是 `"def"`，它们不是同一个池子
- 每一层都是从**当前数字对应的全新字母串**中全量选择
- 不存在「回头选导致重复」的问题，因为每一层的字母串都是独立的

```cpp
// 电话号码问题：每一层的候选集是当前数字对应的字母串
string digit = phoneMap[digits[start]];  // 比如 "abc" 或 "def"
for (int i = 0; i < digit.length(); i++) {  // ← 从0开始，遍历整个字母串
    combination.push_back(digit[i]);
    backtracing(combinations, combination, digits, start + 1);  // 处理下一个数字
    combination.pop_back();
}
```

### 3.3 对比图解

**组合问题（LeetCode 77）**：
```
候选集：[1, 2, 3, 4]（同一个池子）

        选1(start=1)
       /    |    \
     选2   选3   选4   ← i 从 start=1 开始，只能往后
    /  |    |     
   选3 选4  选4    ← i 从 i+1 开始
```

**电话号码问题（LeetCode 17）**：
```
digits = "23"

第1层：数字 '2' → 候选集 "abc"（全新池子）
            a        b        c
           /        /        /
第2层：数字 '3' → 候选集 "def"（全新池子）
         / | |    / | |    / | |
        d  e f   d  e f   d  e f
        ↓  ↓ ↓   ↓  ↓ ↓   ↓  ↓ ↓
       ad ae af  bd be bf  cd ce cf
```

**每一层都是全新的字母串，所以每次都从 0 开始遍历！**

### 3.4 一句话总结

> **组合问题**：所有层共享同一个候选集，需要 `start` 防重复回头。  
> **电话号码问题**：每一层有独立的候选集（不同数字对应不同字母），每次都从头遍历即可。

---

## ❓ 四、问题二：为什么 `unordered_map` 不能放在函数里？

### 4.1 你的代码结构

```cpp
class Solution {
    // ✅ 正确：定义在类级别，所有成员函数都能访问
    unordered_map<char, string> phoneMap{...};

public:
    vector<string> letterCombinations(string digits) {
        // ❌ 错误：如果放在这里...
        // unordered_map<char, string> phoneMap{...};

        backtracing(...);  // 回溯函数在类外面/下面，访问不到局部变量
    }

private:
    void backtracing(...) {
        string digit = phoneMap[digits[start]];  // 需要访问 phoneMap
    }
};
```

### 4.2 C++ 作用域规则

| 定义位置 | 作用域 | 能否被 `backtracing` 访问 |
|---------|--------|------------------------|
| **类成员变量**（你的写法） | 整个 `Solution` 类 | ✅ 可以 |
| `letterCombinations` **函数内**（局部变量） | 仅该函数内部 | ❌ 不可以 |
| `backtracing` **参数传递** | 仅该函数内部 | ✅ 可以（但需要传参） |

### 4.3 三种解决方案

**方案一：类成员变量（你的写法，推荐）**

```cpp
class Solution {
    unordered_map<char, string> phoneMap{...};  // 类成员
public:
    vector<string> letterCombinations(string digits) {
        backtracing(...);  // backtracing 可以直接访问 phoneMap
    }
private:
    void backtracing(...) {
        string digit = phoneMap[digits[start]];  // ✅ 直接访问
    }
};
```

**方案二：作为参数传递**

```cpp
class Solution {
public:
    vector<string> letterCombinations(string digits) {
        unordered_map<char, string> phoneMap{...};  // 局部变量
        backtracing(..., phoneMap);  // 传引用给回溯函数
    }
private:
    void backtracing(..., unordered_map<char, string>& phoneMap) {  // 接收引用
        string digit = phoneMap[digits[start]];  // ✅ 通过参数访问
    }
};
```

> 注意：传引用 `&` 避免拷贝，提高效率。

**方案三：定义为静态局部变量（不推荐，但可行）**

```cpp
void backtracing(...) {
    static unordered_map<char, string> phoneMap{...};  // 静态局部变量
    string digit = phoneMap[digits[start]];
}
```

> 静态局部变量只初始化一次，生命周期贯穿整个程序。但语义不清晰，不推荐。

### 4.4 最佳实践建议

对于回溯类题目：
- **不变的数据**（如映射表、固定数组）→ 定义为**类成员变量**
- **变化的状态**（如当前路径、已访问标记）→ 作为**函数参数或成员变量**
- **避免不必要的参数传递**：类成员变量所有函数共享，代码更简洁

---

## 💻 五、完整代码（优化版）

```cpp
class Solution {
    // 数字到字母的映射表（类成员，所有函数共享）
    unordered_map<char, string> phoneMap{
        {'2', "abc"},
        {'3', "def"},
        {'4', "ghi"},
        {'5', "jkl"},
        {'6', "mno"},
        {'7', "pqrs"},
        {'8', "tuv"},
        {'9', "wxyz"}
    };

public:
    vector<string> letterCombinations(string digits) {
        vector<string> combinations;

        // 边界情况
        if (digits.empty()) return combinations;

        string combination;
        backtracing(combinations, combination, digits, 0);
        return combinations;
    }

private:
    void backtracing(vector<string>& combinations, string& combination, 
                     const string& digits, int start)
    {
        // 终止条件：所有数字都处理完毕
        if (start == digits.length()) {
            combinations.push_back(combination);
            return;
        }

        // 取出当前数字对应的字母串
        const string& letters = phoneMap[digits[start]];

        // 遍历当前数字对应的每一个字母（从0开始！）
        for (int i = 0; i < letters.length(); i++) {
            combination.push_back(letters[i]);      // 选择
            backtracing(combinations, combination, digits, start + 1);  // 递归处理下一个数字
            combination.pop_back();                  // 撤销选择（回溯）
        }
    }
};
```

### 代码优化点

| 优化 | 说明 |
|-----|------|
| `string& combination` | 传引用，避免拷贝 |
| `const string& digits` | 传 const 引用，避免拷贝且不可修改 |
| `const string& letters` | 避免拷贝映射表中的字符串 |
| 空字符串判断 | `if (digits.empty())` 提前返回 |

---

## 🌳 六、递归树可视化（以 digits = "23" 为例）

```
                    开始 ""
                   /   |   \
                 a     b     c          ← 第1层：数字 '2' → "abc"，i 从 0 到 2
               / | | / | | / | |
              d  e f d e f d e f       ← 第2层：数字 '3' → "def"，i 从 0 到 2
              ↓  ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓
            "ad" "ae" "af" "bd" "be" "bf" "cd" "ce" "cf"
```

**执行流程**：

| 步骤 | 当前数字 | 候选字母 | 选择 | combination | 说明 |
|-----|---------|---------|------|------------|------|
| 1 | '2' (start=0) | "abc" | a | "a" | 第1层选 a |
| 2 | '3' (start=1) | "def" | d | "ad" | 第2层选 d |
| 3 | 无 (start=2) | — | — | "ad" | ✅ 加入结果 |
| 4 | 回溯 | — | — | "a" | pop d，回到第2层 |
| 5 | '3' (start=1) | "def" | e | "ae" | 第2层选 e |
| 6 | 无 (start=2) | — | — | "ae" | ✅ 加入结果 |
| ... | ... | ... | ... | ... | 以此类推 |

---

## 📊 七、复杂度分析

- **时间复杂度**：`O(3^m × 4^n)`
  - `m` 是对应 3 个字母的数字个数（2,3,4,5,6,8）
  - `n` 是对应 4 个字母的数字个数（7,9）
  - 每个数字的选择数相乘，即笛卡尔积的大小

- **空间复杂度**：`O(m + n)`
  - 递归栈深度等于输入字符串长度（最多 4）
  - 加上 `combination` 字符串

---

## ⚔️ 八、核心对比：组合 vs 电话号码问题

| 对比维度 | **组合（LeetCode 77）** | **电话号码（LeetCode 17）** |
|---------|------------------------|---------------------------|
| **候选集** | 同一池子 `[1..n]` | 每一层独立（不同数字对应不同字母） |
| **`start` 含义** | 「从哪个数开始选」（防重复） | 「处理到第几个数字」 |
| **`for` 起点** | `i = start`（只能往后） | `i = 0`（每次都从头遍历当前字母串） |
| **递归参数** | `backtrack(i + 1)` | `backtrack(start + 1)` |
| **终止条件** | `pack.size() == k` | `start == digits.length()` |
| **防重机制** | `start` 参数限制范围 | 天然不需要（每层候选集不同） |
| **回溯操作** | `pop_back()` | `pop_back()` |
| **类比** | 从一个篮子里挑球 | 从多个不同篮子里各挑一个球 |

---

## 🚀 九、举一反三：多层选择类题单

| 题目 | 难度 | 核心特征 | 与 17 的相似点 |
|-----|------|---------|--------------|
| **17. 电话号码** | 中等 | 数字→字母映射 | 每层候选集独立，都从 0 遍历 |
| **77. 组合** | 中等 | 从 n 个数选 k 个 | 同一候选集，需要 start 防重 |
| **78. 子集** | 中等 | 所有子集 | 同一候选集，每个节点都是答案 |
| **39. 组合总和** | 中等 | 数组 + 目标和 | 同一候选集，可重复选 |
| **46. 全排列** | 中等 | 所有排列 | 同一候选集，顺序不同算不同 |
| **131. 分割回文串** | 中等 | 字符串分割 | 每层从当前位置开始切分 |
| **93. 复原 IP 地址** | 中等 | IP 地址分段 | 每层从当前位置开始取 1-3 位 |

---

## 💡 十、学习建议

1. **理解 `start` 的双重身份**
   - 在组合/子集问题中：`start` = 「候选集的起始下标」（防重）
   - 在电话号码/分割问题中：`start` = 「处理到输入的第几个位置」（进度）

2. **判断 `for` 起点的方法**
   - 如果所有层共享**同一个候选集** → 需要 `start` 防重，从 `start` 开始
   - 如果每层有**独立的候选集** → 不需要防重，从 `0` 开始

3. **类成员 vs 局部变量**
   - 多个函数共享的数据 → 类成员
   - 单个函数专用的数据 → 局部变量（或传参）
   - 回溯函数的共享数据 → 优先考虑类成员，减少参数冗余

---

## 🎯 一句话总结

> **电话号码问题中，`start` 是「处理到第几个数字」的进度标记，不是「候选集起始位置」。每一层的候选集（字母串）都是全新的，所以 `for` 循环每次都从 0 开始遍历。**
>
> **`unordered_map` 必须定义为类成员变量，因为 C++ 的局部变量只在定义它的函数内可见，回溯函数无法访问 `letterCombinations` 函数内部的局部变量。**

---

> 祝你刷题顺利，早日掌握回溯的精髓！🎉
