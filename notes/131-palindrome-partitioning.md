# 力扣 131：分割回文串 — 学习笔记

---

## 一、题目概述

**给定**：字符串 `s`  
**求**：将 `s` 分割成若干子串，使每个子串都是**回文串**，返回所有可能的分割方案。

**示例**：`s = "aab"`  
**输出**：`[["a","a","b"], ["a","ab"], ["aa","b"]]`

---

## 二、思路分析

### 2.1 问题转化

这本质上是一个**组合/分割问题**：从字符串的第一个字符开始，决定"在哪里切一刀"，切出来的每一段都必须是回文。

### 2.2 回溯设计

**核心思想**：从左到右遍历，维护一个"当前正在构建的子串"，每遇到一个字符有两种选择：

| 选择 | 含义 | 操作 |
|------|------|------|
| **不切割** | 把当前字符拼到上一个子串末尾 | 继续往后走，`sub += s[i]` |
| **切割** | 如果当前子串是回文，就把它作为一段 | 加入 `path`，从下一个位置重新开始新子串 |

**递归终止**：走到字符串末尾，收集当前路径。

---

## 三、代码逐行解析

```cpp
class Solution {
public:
    vector<vector<string>> ans;   // 存储所有分割方案
    vector<string> path;          // 当前正在构建的方案

    // 判断字符串是否为回文
    bool isPalindrome(string& str) {
        int left = 0;
        int right = str.size() - 1;
        while(left < right) {
            if(str[left++] != str[right--]) {
                return false;
            }
        }
        return true;
    }

    /**
     * 回溯函数
     * @param i   当前处理到原字符串的第 i 个字符
     * @param s   原字符串
     * @param sub 从上一个切割点到当前位置正在构建的子串
     */
    void backtracing(int i, const string& s, const string& sub) {
        // 【终止条件】走到字符串末尾，当前方案完成
        if(i == s.size()) {
            ans.push_back(path);
            return;
        }

        // 把当前字符拼到正在构建的子串后面
        string temp = sub + s[i];

        // 【选择1：不切割】继续扩展当前子串（最后一个字符不能选这个，因为必须切）
        if(i < s.size() - 1) {
            backtracing(i + 1, s, temp);
        }

        // 【选择2：切割】如果当前子串是回文，就切一刀
        if(isPalindrome(temp)) {
            path.push_back(temp);        // 把回文子串加入当前方案
            backtracing(i + 1, s, "");   // 从下一个位置重新开始构建子串
            path.pop_back();             // 回溯，撤销选择
        }
    }

    vector<vector<string>> partition(string s) {
        backtracing(0, s, "");
        return ans;
    }
};
```

---

## 四、推演示例

**输入**：`s = "aab"`

```
backtracing(0, "aab", ""), path=[]
│
├─ i=0, temp="a"
│   ├─ 【不切割】backtracing(1, "aab", "a")
│   │   ├─ i=1, temp="aa"
│   │   │   ├─ 【不切割】backtracing(2, "aab", "aa")
│   │   │   │   └─ i=2, temp="aab" → 不是回文，且 i=2 不切割分支不执行 → 无结果
│   │   │   └─ 【切割】"aa"是回文 → path=["aa"], backtracing(2, "aab", "")
│   │   │       └─ i=2, temp="b" → "b"是回文 → path=["aa","b"] → 收集 ✅
│   │   └─ 【切割】"a"是回文 → path=["a"], backtracing(2, "aab", "")
│   │       └─ i=2, temp="b" → "b"是回文 → path=["a","b"]... 等等
│   │           这里不对，让我重新走
│   
│   重新梳理：
│   backtracing(1, "aab", ""), path=["a"]  ← 注意这里 sub 被重置了
│   ├─ i=1, temp="a"
│   │   ├─ 【不切割】backtracing(2, "aab", "a")
│   │   │   └─ i=2, temp="aa" → "aa"是回文 → path=["a","aa"] → 收集 ✅
│   │   └─ 【切割】"a"是回文 → path=["a","a"], backtracing(2, "aab", "")
│   │       └─ i=2, temp="b" → "b"是回文 → path=["a","a","b"] → 收集 ✅
│   └─ ...
│
└─ 【切割】"a"是回文 → path=["a"], backtracing(1, "aab", "")
    ...（同上）

等等，上面的树画混了。让我重新画：

backtracing(0, "aab", ""), path=[]
│
├─ i=0, temp="a"
│   ├─ 【不切割】backtracing(1, "aab", "a"), path=[]
│   │   ├─ i=1, temp="aa"
│   │   │   ├─ 【不切割】backtracing(2, "aab", "aa"), path=[]
│   │   │   │   └─ i=2, temp="aab" → 不是回文，无结果
│   │   │   └─ 【切割】"aa"是回文 → path=["aa"], backtracing(2, "aab", "")
│   │   │       └─ i=2, temp="b" → "b"回文 → path=["aa","b"] ✅
│   │   └─ 【切割】"a"是回文 → path=["a"], backtracing(2, "aab", "")
│   │       └─ i=2, temp="b" → "b"回文 → path=["a","b"]... 不对
│   │           这里 sub="" 是因为从 i=1 切割后重置的，但 path 里有 "a"
│   │           i=2 时 temp="b"，是回文，path.push_back("b") → ["a","b"]
│   │           然后 backtracing(3...) → 收集 ["a","b"]？
│   │           但 "ab" 不是回文，这里 temp="b" 是因为 sub="" + s[2]="b"
│   │           所以 path=["a","b"] → 但 "ab" 并没有被加入 path！
│   │           等等，这里有问题...
│   │
│   │   实际上从 backtracing(1, "aab", "") 开始：
│   │   i=1, temp="a"（sub="" + s[1]='a'）
│   │   ├─ 不切割：backtracing(2, "aab", "a"), path=["a"]
│   │   │   └─ i=2, temp="aa"（sub="a"+s[2]='a'）
│   │   │       "aa"回文 → path=["a","aa"] ✅
│   │   └─ 切割：path=["a","a"], backtracing(2, "aab", ""), pop后path=["a"]
│   │       └─ i=2, temp="b"
│   │           "b"回文 → path=["a","a","b"] ✅
│   │
│   └─ 【切割】"a"回文 → path=["a"], backtracing(1, "aab", ""), pop后path=[]
│       └─ （同上，从i=1, sub=""开始）
│           最终会得到 ["a","a","b"] 和 ["a","aa"]
│
└─ ... 但这样 "a" 开头的会重复？

不，实际上：
- 从 i=0 不切割分支得到的方案：["aa","b"], ["aab"]（如果aab是回文的话）
- 从 i=0 切割分支（path=["a"]）然后走 i=1 的分支：["a","a","b"], ["a","aa"]

所以最终：
1. ["a","a","b"] — 每字符都切
2. ["a","aa"] — 第一次切，第二次不切
3. ["aa","b"] — 第一次不切，第二次切
```

**最终答案**：`[["a","a","b"], ["a","aa"], ["aa","b"]]` ✅

---

## 五、关键要点

| 要点 | 说明 |
|------|------|
| `sub` 的作用 | 记录从上一个切割点到当前位置正在"攒"的子串 |
| 为什么先不切割、后切割 | 保证所有可能的子串长度都被枚举到 |
| `i < s.size()-1` 的判断 | 最后一个字符必须走"切割"分支，否则无法到达终点收集答案 |
| 为什么切割后 `sub=""` | 切了一刀，下一个子串从头开始攒 |
| 回文判断时机 | 只有在决定"切割"时才判断，不切割时不需要管 |

---

## 六、复杂度分析

| 维度 | 复杂度 | 说明 |
|------|--------|------|
| **时间** | O(n × 2ⁿ) | 每个位置理论上切/不切，共 2ⁿ 种分割方式，每种判断回文 O(n) |
| **空间** | O(n) | 递归栈深度 + 当前路径，最多 n 层 |

---

## 七、一句话记忆

> **从左到右攒子串，攒一段判断一下是不是回文，是就切一刀从头攒，不是就继续往后攒；走到末尾收集方案。**
