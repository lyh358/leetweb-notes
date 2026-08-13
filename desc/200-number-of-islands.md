# 200.岛屿的数量

# 题目简述

给你一个由 `'1'`（陆地）和 `'0'`（水）组成的的二维网格，请你计算网格中岛屿的数量。

岛屿总是被水包围，并且每座岛屿只能由水平方向和/或竖直方向上相邻的陆地连接形成。

此外，你可以假设该网格的四条边均被水包围。

**示例 1：**

```
输入：grid = [
  ['1','1','1','1','0'],
  ['1','1','0','1','0'],
  ['1','1','0','0','0'],
  ['0','0','0','0','0']
]
输出：1
```

**示例 2：**

```
输入：grid = [
  ['1','1','0','0','0'],
  ['1','1','0','0','0'],
  ['0','0','1','0','0'],
  ['0','0','0','1','1']
]
输出：3
```

注意题目中每座岛屿只能由**水平方向和/或竖直方向上**相邻的陆地连接形成。

也就是说斜角度链接是不算了， 例如示例二，是三个岛屿，如图：

![](https://pic.leetcode.cn/1658888882-OXiONL-image.png)

---
```
class Solution {
public:
    int dir[4][2]={{0,1},{1,0},{0,-1},{-1,0}};
    int count=0;
    void dfs(vector<vector<char>>& grid,vector<vector<int>>& visited,int x,int y)
    {
        for(int i=0;i<4;i++)
        {
            int nextx=x+dir[i][0];
            int nexty=y+dir[i][1];
            if(nextx<0 || nextx>=grid.size() || nexty<0 || nexty>=grid[0].size())
            {
                continue;
            } 
            if(grid[nextx][nexty]=='1' && visited[nextx][nexty]==0)
            {
                visited[nextx][nexty]=1;
                dfs(grid,visited,nextx,nexty);
            }     
        }
    }
    int numIslands(vector<vector<char>>& grid) {
        int m=grid.size();
        int n=grid[0].size();
        vector<vector<int>> visited(m,vector<int>(n,0));
        for(int i=0;i<m;i++)
        {
            for(int j=0;j<n;j++)
            {
                if(grid[i][j]=='1' && visited[i][j]==0)
                {
                    visited[i][j]=1;
                    count++;
                    dfs(grid,visited,i,j);
                }
            }
        }
        return count;
    }
};
```