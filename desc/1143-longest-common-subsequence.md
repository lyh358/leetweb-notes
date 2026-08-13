# 力扣 1143. 最长公共子序列 — 学习笔记

> **题目**：给定两个字符串 `text1` 和 `text2`，返回这两个字符串的**最长公共子序列**的长度。如果不存在公共子序列，返回 `0`。  
> **难度**：中等  
> **标签**：字符串、动态规划

---

## 一、什么是"子序列"？

**子序列** = 从原字符串里**挑一些字符**（不改变它们的先后顺序），组成的新字符串。

不要求连续！中间可以跳过字符。

```
原串："abcde"
子序列："ace"    ✅（按顺序挑了 a、c、e，中间跳过了 b、d）
子序列："aec"    ❌（顺序乱了，e 在 c 前面了）
子序列："acd"    ✅（按顺序挑了 a、c、d）
```

**公共子序列** = 两个字符串都拥有的子序列。

```
text1 = "abcde"
text2 = "ace"

公共子序列："ace"    ← 最长的那个，长度是 3
```

---
```
class Solution {
public:
    int longestCommonSubsequence(string text1, string text2) {
        int m=text1.size();
        int n=text2.size();

        vector<vector<int>> dp(m+1,vector<int>(n+1,0));

        for(int i=1;i<=m;i++)
        {
            for(int j=1;j<=n;j++)
            {
                if(text1[i-1]==text2[j-1])
                {
                    dp[i][j]=dp[i-1][j-1]+1;
                }
                else
                {
                    dp[i][j]=max(dp[i-1][j],dp[i][j-1]);
                }
            }
        }
        return dp[m][n];
    }
};
```
