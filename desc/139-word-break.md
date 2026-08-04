# LeetCode 139. 单词拆分（Word Break）

## 1. 题目描述

给定一个字符串 `s` 和一个字符串数组 `wordDict`。

判断 `s` 是否可以被拆分成一个或多个字典中的单词。

注意：

- 字典中的单词可以重复使用。
- 单词之间不需要真的添加空格，只需要判断是否存在一种拆分方式。

例如：

```
输入：
s = "leetcode"
wordDict = ["leet", "code"]

输出：
true

解释：
leetcode = leet + code
```

题目本质：

> 能不能把一个字符串切成若干段，每一段都存在于字典中？

---
```
````md
# C++ STL 范围构造（Range Constructor）

## 基本写法

```cpp
unordered_set<string> dict(
    wordDict.begin(),
    wordDict.end()
);
```

**简单理解：**

> 用 `wordDict` 的全部元素初始化一个 `unordered_set`。

---

## 示例

```cpp
vector<string> wordDict = {"leet", "code"};

unordered_set<string> dict(
    wordDict.begin(),
    wordDict.end()
);
```

等价于：

```cpp
unordered_set<string> dict;

dict.insert("leet");
dict.insert("code");
```

---

## 通用写法

以后看到这种：

```cpp
容器B(容器A.begin(), 容器A.end());
```

基本就是：

> **把容器A里面的所有元素复制/转换到容器B。**

---

## 再举一个例子

```cpp
vector<int> a = {1,2,3};

set<int> b(a.begin(), a.end());
```

等价于：

```cpp
set<int> b = {1,2,3};
```

---

## begin() 和 end() 是什么？

它们都是 **迭代器（Iterator）**。

- `begin()`：指向第一个元素
- `end()`：指向最后一个元素的后一个位置（不是最后一个元素）

表示一个范围：

```text
[begin, end)
```

即：

- ✅ 包含 `begin`
- ❌ 不包含 `end`

---

## 在 LeetCode 中常见的用法

```cpp
vector -> unordered_set
vector -> set
vector -> map
```

都是利用这种**范围构造函数（Range Constructor）**，一次性把一个容器中的所有元素放到另一个容器中。

---

## 一句话记忆

> **用一对迭代器，把一个容器里的所有元素批量复制（或转换）到另一个容器。**
````
```
