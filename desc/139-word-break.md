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
dp[i]是前i个字母是否可以被拆分，
```
class Solution {
public:
    bool wordBreak(string s, vector<string>& wordDict) {
        unordered_set<string> set(wordDict.begin(),wordDict.end());
        int n = s.size();
        vector<bool> dp(n+1);
        dp[0]=true;

        for(int i=1;i<=n;i++)
        {
            for(int j=0;j<i;j++)
            {
                if(dp[j] && set.count(s.substr(j,i-j)))
                {
                    dp[i] = true;
                }
            }
        }
        return dp[n];
    }
};
```
