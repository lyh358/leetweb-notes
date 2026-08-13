# 力扣 72. 编辑距离 — 学习笔记

> **题目**：给你两个单词 `word1` 和 `word2`，请返回将 `word1` 转换成 `word2` 所使用的**最少操作数**。  
> 你可以对一个单词进行如下三种操作：
> - 插入一个字符
> - 删除一个字符
> - 替换一个字符
> 
> **难度**：中等  
> **标签**：字符串、动态规划

---

## 一、题目理解

**示例**：
```
word1 = "horse"
word2 = "ros"

最少操作 3 步：
  horse → rorse  (将 h 替换为 r)
  rorse → rose   (删除 r)
  rose  → ros    (删除 e)

输出: 3
```

**核心问题**：两个字符串不一样，怎么用最少的"插入/删除/替换"把它们变成一样？

---
```
class Solution {
public:
    int minDistance(string word1, string word2) {
        int m = word1.size();
        int n = word2.size();

        if(m*n==0) return m+n;

        vector<vector<int>> dp(m+1,vector<int>(n+1,0));

        for(int i=0;i<=m;i++)
        {
            dp[i][0]=i;
        }
        for(int j=0;j<=n;j++)
        {
            dp[0][j]=j;
        }

        for(int i=1;i<=m;i++)
        {
            for(int j=1;j<=n;j++)
            {
                if(word1[i-1]==word2[j-1])
                {
                    dp[i][j]=dp[i-1][j-1];
                }
                else
                {
                    dp[i][j]=min({dp[i-1][j]+1,dp[i][j-1]+1,dp[i-1][j-1]+1});
                }
            }
        }
        return dp[m][n];
    }
};
```
