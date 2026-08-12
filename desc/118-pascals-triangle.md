# 力扣 118. 杨辉三角 — 学习笔记

> **题目**：给定一个非负整数 `numRows`，生成「杨辉三角」的前 `numRows` 行。  
> **难度**：简单  
> **标签**：数组、动态规划

---

## 一、题目理解

杨辉三角（Pascal's Triangle）的规律：

```
        1           ← 第 0 行
       1 1          ← 第 1 行
      1 2 1         ← 第 2 行
     1 3 3 1        ← 第 3 行
    1 4 6 4 1       ← 第 4 行
```

**核心性质**：
- 每行的首尾元素都是 `1`
- 中间元素 = 上一行左上方元素 + 上一行正上方元素

---
```
class Solution {
public:
    vector<vector<int>> generate(int numRows) {
        //先只分配行数
        vector<vector<int>> ans(numRows);   
        //在要求的行数内构造
        for(int i=0;i<numRows;i++)
        {
            //在外层循环中构造每行的列数：因为三角不是一个矩阵
            ans[i].resize(i + 1);   //  第0行有1个元素
            //初始化边界条件：c[i][0]=c[i][i]=1
            ans[i][0]=1;
            ans[i][i]=1;
            //内层循环通过上层元素构建下层：j-1>0;j是上一行的j_max=i-1所以j<i;
            for(int j=1;j<i;j++)
            {
                //通项公式： c[i][j]=c[i−1][j−1]+c[i−1][j]
                ans[i][j]=ans[i-1][j-1]+ans[i-1][j];
            }
        }
        return ans;
    }
};
```
