# 方法：动态规划

## 问题理解

假设你要爬 `n` 阶楼梯，每次你可以：

- 跨 **1** 步
- 跨 **2** 步

问：有多少种不同的方法可以爬到楼顶？

**例子**：`n = 3`

- 1+1+1
- 1+2
- 2+1

**共 3 种方法**。

------

## 动态规划核心思想

### 1. 找规律（最重要的一步）

想爬到第 `n` 阶，你最后一步只有两种可能：

- 从第 `n-1` 阶跨 **1** 步上来
- 从第 `n-2` 阶跨 **2** 步上来

**所以**：爬到第 `n` 阶的方法数 = 爬到第 `n-1` 阶的方法数 + 爬到第 `n-2` 阶的方法数

![image-20260413224025778](C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20260413224025778.png)

# 滚动变量实现：

```c++
public int climbStairs(int n) {
        // 爬一楼的方法数量
        int p = 1;
       // 爬二楼的方法数量
        int q = 2;
        if(n == 1){
            return p;
        }else if(n == 2){
            return q;
        }else{
            // 从第三楼开始，只有两种上楼方式，从前一层再爬一楼和从前二层再爬两楼。
           // 可以推出 f(n) = f(n -1) + f(n -2)
           // 直接递归会超时，所以用的for循环求结果
            int r = 0;
            for(int i = 3; i <= n; i++){
                r = q + p;
                p = q;
                q = r;
            }
            return r;
        }
    }
```

## 简化版本：

```c++
class Solution {
public:
    int climbStairs(int n) {
        int p = 0, q = 0, r = 1;
        for (int i = 1; i <= n; ++i) {
            p = q; 
            q = r; 
            r = p + q;
        }
        return r;
    }
};
```

## 动态规划通项公式版

```c++
class Solution {
public:
    int climbStairs(int n) {
        //动态规划DP:n的方法数
        //dp[n]=dp[n-1]+dp[n-2]

        vector<int> dp(n+1,0);//为了对应输出的dp[n]

        //及时return 防止n<3的时候越界访问
        if(n==1) return 1;
        if(n==2) return 2;
        dp[0]=0;
        dp[1]=1;
        dp[2]=2;

        

        for(int i=3;i<=n;i++)
        {
            dp[i]=dp[i-1]+dp[i-2];
        }
        return dp[n];
    }
};
```



---



# 变体

## 1.每步可以走1到k个台阶

## 	通用规律

如果每步能走 `1~k` 步，需要 **`k+1` 个变量** 滚动

**初始化：前k个变量是0，最后一个变量是1**