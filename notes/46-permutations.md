# 核心方法：DFS+回溯

## 主要思路：把数组切成两半

```plain
output = [已确定的部分 | 待排列的部分]
          ↑first           ↑first到len-1
```

- `first`：分界线，左边是已经填好的位置
- `first`到`len-1`：还没填，需要枚举所有可能

# part1:DFS回溯函数

## (1) DFS回溯函数有四个形参：

- #### 二维动态数组res：用来储存最终的组合结果

- #### 一维动态数组output：用于组合每一个数字排列

- #### first：用于切割output中的已排列部份和未排列部分first到len-1

- #### len：输入的待排列数组的长度

```c++
void DFS_Traceback(vector<vector<int>>& res, vector<int>& output,int first,int len)
{
    
}
```

---

## (2) 递归结束条件：first与len重合，即所有数都已经被排列过

- #### 此时将当前的output存入res中并返回到上一层

```c++
void DFS_Traceback(vector<vector<int>>& res, vector<int>& output,int first,int len)
{
    if(first==len)
    {
        res.push_back(output);
        return;
    }
}
```

---

## (3) DFS主流程

- ### 循环递归+回溯

---

### 以nums=[1，2，3]为例：first最开始为0

### for(int i = first;i<len;i++)：

- ### 第一层循环，就是把[1,X,X]全排列一遍

  - #### 每一层中先进行一次output[i]和output[first]的交换，探索新组合==（swap）==

  - #### 每一层内部的==递归==就是保持头不变，交换后面的数==（通过传入first+1实现）==

  - #### ==回溯==：再进行一次output[i]和output[first]的交换，恢复到原状态，以便于进行下一轮循环

- ### 第二层循环，就是把[2,X,X]全排列一遍

- ### 第三层循环，就是把[3,X,X]全排列一遍

----

### 循环结束：所有组合都已存入res中

```c++
void DFS_Traceback(vector<vector<int>>& res, vector<int>& output,int first,int len)
{
    //递归结束条件
    if(first==len)
    {
        res.push_back(output);
        return;
    }
    //DFS主流程
    for(int i = first;i<len;i++)
    {
        swap(output[i],output[first]);
        DFS_Traceback(res, output, first+1, len);
    }
}
```

---



# part2：主函数

- ### 初始化res数组

- ### 调用DFS回溯函数

  - #### 传入res

  - #### 传入题目中给的数组nums作为output

  - #### 传入first=0

  - #### 传入len=nums.size()

- ### 返回res结果

```c++
 vector<vector<int>> permute(vector<int>& nums) 
 {
     	//初始化res数组
        vector<vector<int> > res;
     	//调用DFS回溯函数
        backtrack(res, nums, 0, (int)nums.size());
     	//返回res结果
        return res;
  }
```

---



# 完整代码

```c++
class Solution {
public:
    //part1:DFS函数
    void backtrack(vector<vector<int>>& res, vector<int>& output, int first, int len){
        // 所有数都填完了
        if (first == len) 
        {
            res.push_back(output);
            return;
        }
        for (int i = first; i < len; ++i) 
        {
            // 动态维护数组
            swap(output[i], output[first]);
            // 继续递归填下一个数
            backtrack(res, output, first + 1, len);
            // 撤销操作
            swap(output[i], output[first]);
        }
    }
    
    //part2:主函数
    vector<vector<int>> permute(vector<int>& nums) 
    {
        vector<vector<int> > res;
        backtrack(res, nums, 0, (int)nums.size());
        return res;
    }
};
```

