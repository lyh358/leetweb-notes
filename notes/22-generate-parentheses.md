# 🌲 回溯算法学习笔记 —— 力扣 22. 括号生成（C++）

> 基于「选/不选」模型的显式回溯解法，总结所有踩过的坑

---

## 📌 一、题目：LeetCode 22. 括号生成

**题目链接**：[22. 括号生成](https://leetcode.cn/problems/generate-parentheses/)

**题目描述**：数字 `n` 代表生成括号的对数，请你设计一个函数，用于能够生成所有可能的并且**有效的**括号组合。

**示例**：

```
输入: n = 3
输出: ["((()))","(()())","(())()","()(())","()()()"]

输入: n = 1
输出: ["()"]
```

**约束**：
- `1 <= n <= 8`

---

## 💻 二、标准解法（你的最终代码）

```cpp
class Solution {
public:
    vector<string> ans;
    string str = "";

    void dfs(string& str, int left, int right)
    {
        // 剪枝条件：已放右括号比左括号多，一定非法
        if (left > right) return;

        // 终止条件：左右括号都用完了，得到一个合法组合
        if (left == 0 && right == 0) {
            ans.push_back(str);
            return;
        }

        // 分支 1：放左括号（前提：还有剩余）
        if (left > 0) {
            str.push_back('(');           // 做选择
            dfs(str, left - 1, right);    // 递归
            str.pop_back();               // 撤销选择（回溯）
        }

        // 分支 2：放右括号（前提：还有剩余）
        if (right > 0) {
            str.push_back(')');           // 做选择
            dfs(str, left, right - 1);    // 递归
            str.pop_back();               // 撤销选择（回溯）
        }
    }

    vector<string> generateParenthesis(int n) {
        dfs(str, n, n);
        return ans;
    }
};
```

---

## 🧠 三、核心思路解析

### 3.1 参数设计

| 参数 | 类型 | 含义 |
|-----|------|------|
| `str` | `string&` | 当前已构造的括号字符串（引用传递，原地修改） |
| `left` | `int` | **剩余**还可以放的左括号数量 |
| `right` | `int` | **剩余**还可以放的右括号数量 |

初始调用：`dfs(str, n, n)` —— 左右括号各剩 `n` 个可用。

### 3.2 合法括号的铁律

1. **数量相等**：最终 `'('` 和 `')'` 的数量都等于 `n`
2. **前缀合法**：构造过程中任意前缀中，`'('` 的数量 ≥ `')'` 的数量

### 3.3 为什么用 `string&` + `push_back/pop_back`？

这是**显式回溯**的写法：
- `push_back('(')` → 把 `'('` 加入当前字符串
- `dfs(...)` → 基于新字符串递归
- `pop_back()` → **撤销**刚才的选择，恢复现场

> 如果不 `pop_back()`，字符串只会越来越长，状态完全混乱。

---

## ⚠️ 四、踩坑记录：所有错误及原因

### 坑 1：引用传递 + 原地修改，但忘记回溯

**错误代码**：
```cpp
void dfs(string& str, int left, int right) {
    // ...
    dfs(str += '(', left - 1, right);   // ← str 被原地修改了！
    dfs(str += ')', left, right - 1);   // ← 在已被修改的 str 上继续加！
}
```

**问题**：`str += '('` 原地修改了字符串，递归返回后没有撤销，导致 `str` 只增不减。

**后果**：字符串长度很快超过 `2n`，左右括号数量完全错乱，结果全是垃圾。

**修复**：要么用 `const string&` + `str + '('`（隐式回溯），要么用 `string&` + `push_back/pop_back`（显式回溯）。

---

### 坑 2：`push_back` 返回 `void`，不能直接传参

**错误代码**：
```cpp
dfs(str.push_back('('), left - 1, right);
// 编译错误！push_back 返回 void，不能传给 string&
```

**问题**：`std::string::push_back` 的返回值是 **`void`**，而 `dfs` 第一个参数要求是 `string&`。

**修复**：拆成两步：
```cpp
str.push_back('(');        // 第一步：修改
dfs(str, left - 1, right); // 第二步：递归
str.pop_back();            // 第三步：回溯
```

---

### 坑 3：两个分支都写成了 `'('`

**错误代码**：
```cpp
str.push_back('('); dfs(str, left - 1, right); str.pop_back();
str.push_back('('); dfs(str, left, right - 1); str.pop_back();  // ← 应该是 ')'！
```

**问题**：第二个分支也 `push_back('(')`，意味着永远只放左括号，右括号分支被写错。

**后果**：只生成 `"(((..."` 这种无限左括号的串，永远凑不出合法组合。

**修复**：第二个分支改成 `push_back(')')`。

---

### 坑 4：缺少 `left < 0` 的防护，导致无限递归

**错误代码**：
```cpp
void dfs(string& str, int left, int right) {
    if (left > right) return;  // ← 只有这一个剪枝
    // ...
    str.push_back('(');
    dfs(str, left - 1, right);  // left 可能变成 -1, -2, ...
    str.pop_back();
    // ...
}
```

**问题**：当 `left` 变成负数后（比如 -1），`left > right`（-1 > 1）为 false，剪枝不触发，递归无限进行，最终栈溢出。

**根因**：代码直接 `push_back('(')` 然后 `left - 1`，没有判断 `left` 是否还有剩余。

**修复方案 A**：加 `left < 0` 剪枝
```cpp
if (left < 0 || left > right) return;
```

**修复方案 B（推荐）**：`push` 前判断
```cpp
if (left > 0) {  // 有剩余才放
    str.push_back('(');
    dfs(str, left - 1, right);
    str.pop_back();
}
```

> 💡 方案 B 更优雅，因为它**根本不会进入无效分支**，而不是进去了再剪枝。

---

### 坑 5：混淆 `const string&` 和 `string&`

| 写法 | 参数类型 | 修改方式 | 是否需要显式回溯 |
|-----|---------|---------|---------------|
| **隐式回溯** | `const string& str` | `str + '('`（创建新串） | ❌ 不需要 |
| **显式回溯** | `string& str` | `str.push_back('(')`（原地改） | ✅ 必须 `pop_back()` |

**错误**：混用两种模式——用 `string&`（引用，可修改）但传 `str + '('`（临时对象），或者反过来用 `const string&` 但尝试 `push_back`。

**原则**：
- 用 `const string&` → 只能读，用 `str + '('` 创建新串传下去
- 用 `string&` → 可以改，但必须 `push_back` + `pop_back` 配对

---

## 🌳 五、递归树可视化（以 n = 3 为例）

```
                              "", left=3, right=3
                             /                  \
                       放 '('                    放 ')'
                      /                          \
              "(", l=2,r=3                  [left=3>right=2! 剪枝] ❌
             /          \
           放 '('        放 ')'
          /              \
    "((", l=1,r=3    "()", l=2,r=2
    /        \
  放 '('     放 ')'
  /          \
"((("l=0,r=3 "(()"l=1,r=2
 |      /        \
放')'  放'('     放')'
...   ...        ...

合法结果（叶子节点）：
"((()))"  "(()())"  "(())()"  "()(())"  "()()()"
```

**关键剪枝点**：
- 根节点直接放 `')'` → `left=3 > right=2` → 剪枝！
- 任何时刻如果已放 `')'` 比 `'('` 多 → 剪枝！

---

## ⚔️ 六、两种正确写法对比

### 写法一：显式回溯（你的最终代码，推荐理解回溯本质）

```cpp
void dfs(string& str, int left, int right) {
    if (left > right) return;
    if (left == 0 && right == 0) { ans.push_back(str); return; }

    if (left > 0) {
        str.push_back('(');
        dfs(str, left - 1, right);
        str.pop_back();  // ← 显式回溯
    }
    if (right > 0) {
        str.push_back(')');
        dfs(str, left, right - 1);
        str.pop_back();  // ← 显式回溯
    }
}
```

**特点**：
- 原地修改字符串，空间效率高
- 必须 `push_back` 和 `pop_back` 配对
- 更直观展示「选-探-撤」的回溯过程

### 写法二：隐式回溯（const 引用 + 新串）

```cpp
void dfs(const string& str, int left, int right) {
    if (left > right) return;
    if (left == 0 && right == 0) { ans.push_back(str); return; }

    if (left > 0) dfs(str + '(', left - 1, right);
    if (right > 0) dfs(str + ')', left, right - 1);
}
```

**特点**：
- 每次递归创建新字符串，代码简洁
- 无需显式回溯，递归返回后原串不变
- 空间开销稍大（创建临时对象）

---

## 📊 七、复杂度分析

- **时间复杂度**：`O(4ⁿ / √n)`
  - 第 `n` 个**卡特兰数（Catalan Number）**的渐近界
  - 卡特兰数：`Cₙ = (1/(n+1)) × C(2n, n)`
  - `n=3` 时 `C₃ = 5`；`n=8` 时 `C₈ = 1430`

- **空间复杂度**：`O(n)`
  - 递归栈深度最多 `2n`
  - 加上存储结果的 `ans` 数组

---

## 🎯 八、回溯口诀（针对本题）

> **有剩余，才能放；**
> **放完了，要撤销；**
> **右比左多，直接剪；**
> **左右用完，就收集。**

---

## 📎 附录：错误代码进化史

```cpp
// ===== 版本 1：原地修改但忘记回溯 =====
void dfs(string& str, int left, int right) {
    dfs(str += '(', left - 1, right);   // ❌ str 被改，没恢复
    dfs(str += ')', left, right - 1);   // ❌ 在已改的 str 上继续改
}

// ===== 版本 2：push_back 返回值当参数 =====
dfs(str.push_back('('), left - 1, right);  // ❌ push_back 返回 void

// ===== 版本 3：两个分支都写 '(' =====
str.push_back('('); ...
str.push_back('('); ...  // ❌ 第二个应该是 ')'

// ===== 版本 4：缺少 left>0 判断，导致 left 为负无限递归 =====
str.push_back('(');
dfs(str, left - 1, right);  // ❌ left 可能变成 -1, -2...

// ===== 版本 5（最终正确）：显式回溯 + 前置判断 =====
if (left > 0) {
    str.push_back('(');
    dfs(str, left - 1, right);
    str.pop_back();  // ✅ 撤销
}
if (right > 0) {
    str.push_back(')');
    dfs(str, left, right - 1);
    str.pop_back();  // ✅ 撤销
}
```

---

> 祝你刷题顺利，踩过的坑都变成经验！🎉
