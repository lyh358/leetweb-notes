# 2. 思路分析

## 暴力思考

假设：

```
s = "leetcode"
```

我们可以尝试：

```
l + eetcode
le +etcode
lee + tcode
leet + code
```

发现：

```
leet 在字典中
code 在字典中
```

所以成功。

但是如果字符串很长：

```
aaaaaaaaaaaaaaaa
```

每个位置都有很多可能切法。

暴力搜索会产生大量重复计算。

例如：

```
abcxxxxx
```

很多路径都会重复判断：

```
xxxxx 能不能拆？
```

所以需要动态规划。

---

# 3. 动态规划思想

## 关键问题

定义：

```
dp[i]
```

表示：

> 字符串 s 的前 i 个字符，是否可以拆分成功。

注意：

这里的 i 表示长度。

例如：

```
s = "leetcode"

dp[4]
```

表示：

```
"leet"
```

是否可以拆分。

---

# 4. 状态定义

数组：

```cpp
vector<bool> dp(s.size()+1,false);
```

长度：

```
s.size()+1
```

为什么多一个？

因为：

```
dp[0]
```

表示：

空字符串是否可以拆分。

空字符串：

```
什么都没有
```

当然可以。

所以：

```cpp
dp[0]=true;
```

---

# 5. 状态转移

我们假设：

```
dp[i]
```

想知道是否成立。

那么我们需要找到一个位置：

```
j
```

把字符串分成：

```
前面部分 + 后面单词
```

例如：

```
leetcode

j        i
|--------|---|
leet     code
```

如果：

```
dp[j]==true
```

并且：

```
s[j,i]
```

在字典里面。

那么：

```
dp[i]=true
```

公式：

```
dp[i] = dp[j] && s[j:i]存在于字典
```

---

# 6. 举例推导


字符串：

```
leetcode
```

字典：

```
["leet","code"]
```


初始化：

```
dp:

index:
0 1 2 3 4 5 6 7 8

value:
T F F F F F F F F
```


---

## i = 4


检查：

```
s[0:4]

leet
```

存在。


因为：

```
dp[0]=true
```

所以：

```
dp[4]=true
```


状态：

```
T F F F T F F F F
```


---

## i = 8


检查：

```
s[4:8]

code
```


发现：

```
dp[4]=true
```

并且：

```
code存在
```


所以：

```
dp[8]=true
```


最终：

```
dp[8]=true
```

答案：

```
true
```

---

# 7. C++代码


```cpp
class Solution {
public:

    bool wordBreak(string s, vector<string>& wordDict) {

        // 使用哈希表提高查找速度
        unordered_set<string> dict(
            wordDict.begin(),
            wordDict.end()
        );


        int n = s.size();


        // dp[i]表示前i个字符是否可以拆分
        vector<bool> dp(n + 1, false);


        // 空字符串可以拆分
        dp[0] = true;


        for(int i = 1; i <= n; i++)
        {

            for(int j = 0; j < i; j++)
            {

                // 如果前面的字符串可以拆
                // 并且当前这一段在字典中

                if(dp[j] &&
                   dict.count(s.substr(j, i-j)))
                {
                    dp[i] = true;
                    break;
                }

            }

        }


        return dp[n];
    }
};
```

---

# 8. 代码解释


## 创建字典


```cpp
unordered_set<string> dict(
    wordDict.begin(),
    wordDict.end()
);
```

作用：

把数组转换成哈希表。


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


原来：

```
vector
```

查找：

```
O(n)
```


转换后：

```
unordered_set
```

查找：

```
O(1)
```


---

## 双层循环


外层：

```cpp
i
```

表示：

```
当前判断到哪里
```


内层：

```cpp
j
```

表示：

```
从哪里切一刀
```


例如：

```
leetcode

j       i
|-------|
leet
```

---

# 9. 时间复杂度


设：

```
n = s长度
```


两层循环：

```
O(n²)
```


每次：

```cpp
substr()
```

需要复制字符串：

```
O(n)
```


所以严格来说：

```
O(n³)
```


但是由于题目限制：

```
s.length <= 300
```

可以通过。


空间：

```
O(n)
```


---

# 10. 常见错误


## 错误1：dp数组长度写错


错误：

```cpp
vector<bool> dp(n,false);
```


应该：

```cpp
vector<bool> dp(n+1,false);
```


因为需要：

```
dp[0]
```


---

## 错误2：忘记初始化


错误：

```cpp
dp[0]=false;
```


导致：

第一个单词永远无法匹配。


正确：

```cpp
dp[0]=true;
```


---

## 错误3：只判断最后一个单词


错误思路：

```
leetcode

直接找leetcode
```


但是答案可能：

```
leet + code
```


所以必须保存之前状态。

---

# 11. 思维总结


这类题看到：

```
字符串 + 字典 + 判断能否组成
```

想到：

```
动态规划
```


核心模板：


```
dp[i]

表示：

前i个字符是否满足条件


状态转移：

寻找 j

dp[j] && j到i这一段满足条件

=> dp[i]=true
```


---

# 12. 类似题目


## LeetCode 140 单词拆分 II

区别：

139：

```
只判断能不能拆
```

140：

```
返回所有拆法
```


---

## LeetCode 322 零钱兑换

类似：

```
dp[i]
```

表示：

达到金额 i 是否可能。

---

## LeetCode 139 本质

一句话：

> 前面的路走通了，后面的单词才能接上。

动态规划就是记录：

```
哪些位置已经可以到达
```

---

# 面试回答模板


如果面试官问：

"怎么解决？"


可以回答：


> 我定义 dp[i] 表示字符串前 i 个字符是否可以被拆分。
> 初始化 dp[0]=true。
> 对每个位置 i，枚举之前的位置 j，
> 如果 dp[j] 为 true，并且 s[j:i] 在字典中，
> 那么 dp[i]=true。
> 最后返回 dp[n]。


这样可以完整表达：

- 状态定义
- 初始化
- 转移方程
- 复杂度分析


---

## 最终记忆口诀

```
单词拆分139：

看到字符串切割问题

定义dp到哪里成功

前面成功 + 当前单词存在

当前位置成功
```
```` citeturn0search1