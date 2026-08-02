# 核心算法：DFS、BFS、并查集

# 核心思路：

#### 每遇到一个==没有遍历过的节点陆地，计数器就加一==，然后把==该节点陆地所能遍历到的陆地都标记上==。

#### 在==遇到标记过的陆地节点和海洋节点的时候直接跳过==。 这样计数器就是最终岛屿的数量。

#### 那么如果把节点陆地所能遍历到的陆地都标记上呢，就==可以使用 DFS，BFS或者并查集==。

---

# DFS思路

- ## 传入grip、访问二维数组，起始坐标

- ## 从初始位置开始向四个方向移动

  - ### 得到新坐标

- ## 越界处理当作结束条件

- ## 若不越界，检查当前块是否为陆地且未被访问过

  - ### 标记为访问

  - ### 递归调用DFS，传入新坐标

# 主函数思路

- ## 构建访问数组，标记为全false

- ## 构建计数变量

- ## 循环访问二维坐标

  - ### 如果   未访问   且  是陆地，计数+1

  - ### 对每一个坐标调用DFS

---



# DFS解法

## 1.构建方向数组，代替四个if，可以往四个方向走

```c++
int dir[4][2]={0,1,
               1,0,
               -1,0,
               0,-1};
```

| 索引     | 含义 | 坐标变化  |
| :------- | :--- | :-------- |
| `dir[0]` | 右   | `(0, +1)` |
| `dir[1]` | 下   | `(+1, 0)` |
| `dir[2]` | 上   | `(-1, 0)` |
| `dir[3]` | 左   | `(0, -1)` |

## 2.构建dfs函数：把连在一起的陆地都标记为已访问

- ### 返回值：void

- ### 参数：

  - #### 二维动态char数组grip引用：地图

  - #### 二维动态bool数组visited引用：访问标记地图

  - #### int x,int y:当前坐标

```c++
void dfs(vector<vector<char>>& grid, vector<vector<int>>& visited, int x, int y)
{
    
}
```

## 3.dfs函数逻辑

#### 1.尝试四个方向：从当前位置 `(x, y)`，往四个方向走一格。

#### 2.越界控制->直接跳过这步

#### 3.判断是否访问过，且是否为陆地(如果满足则将visit[nextx] [nexty]标记为true)

#### 4.递归，继续调用dfs： dfs(grid, visited, nextx, nexty);

```c++
void dfs(vector<vector<char>>& grid, vector<vector<int>>& visited, int x, int y)
{
    for(int i=0;i<4;i++)
    {
        //保证每次只走一格
        int nextx = x + dir[i][0];// 下一个位置的x坐标
        int nexty = y + dir[i][1];// 下一个位置的y坐标
        
        //越界处理
    	if(nextx<0||nextx>=grid.size()||nexty<0||nexty>=grid[0].size())
    	{
            // 越界了，直接跳过（比如走到地图外面了）
        	continue;
    	}
        // 没访问过 且 是陆地
		if( !visited[nextx][nexty] && grip[nextx][nexty]=='1' )
    	{
        	// 将该位置标记为已访问
        	visited[nextx][nexty] = true;
        
        	//递归：继续向四周扩散
        	dfs(grid, visited, nextx, nexty);
    	}
    }
}
```

#### **递归思想**：找到一个陆地，标记它，然后从它继续往四周找，直到周围都是海水或边界为止。



## 4.主函数

- ## int numIslands(vector<vector<char>>& grid)

- ### 用于数岛屿

### 1.得到地图grip的行（n）和列（m）

- **获取地图大小。**

```c++
int n = grid.size(), m = grid[0].size();  // n行 m列
```

### 2.创建访问标记数组：初始化为全false

```c++
 vector<vector<bool>> visited = vector<vector<bool>>(n, vector<bool>(m, false)); 
```

### 3.创建岛屿数量计数器result

```c++
        int result = 0;  // 岛屿数量计数器
```

### 4.用双层循环遍历grip的每一个地块，将每个岛都用dfs填满其visited地图，并统计result

```c++
        for (int i = 0; i < n; i++) {
            for (int j = 0; j < m; j++) {  // 遍历地图每个格子
                if (!visited[i][j] && grid[i][j] == '1') {  
                    // 找到一个没访问过的陆地！
                    visited[i][j] = true;   // 标记
                    result++;               // 岛屿数+1
                    dfs(grid, visited, i, j); // 把这个岛的所有陆地都标记为已访问
                }
            }
        }
        return result;
    }
};
```

---



# DFS完整代码

```c++
class Solution {
private:
    int dir[4][2]={0,1,1,0,-1,0,0,-1};
    void dfs(vector<vector<char>>& grid, vector<vector<bool>>& visited, int x, int y)
    {
        //可选：将当前位置标记为ture
        //因为主函数遍历不会走回头路，所以不标记也不算错
        visited[x][y]=true;
        
        for(int i = 0;i<4;i++)
        {
            int nextx = x + dir[i][0];
            int nexty = y + dir[i][1];

            if(nextx<0||nextx>=grid.size()||nexty<0||nexty>=grid[0].size())
            {
                continue;
            }
            if(grid[nextx][nexty]=='1' && !visited[nextx][nexty])
            {
                visited[nextx][nexty]=true;
                dfs(grid, visited,nextx,nexty);
            }
        }
    }
public:
    int numIslands(vector<vector<char>>& grid)
    {
        int n=grid.size(),m=grid[0].size();
        vector<vector<bool>> visited = vector<vector<bool>>(n,vector<bool>(m,false));
        int result=0;
        for(int i=0;i<n;i++)
        {
            for(int j=0;j<m;j++)
            {
                if(grid[i][j]=='1'&&!visited[i][j])
                {
                    visited[i][j]=true;
                    result++;
                    dfs(grid, visited,i,j);
                }
            }
        }
        return result;
    }
};
```

---

# BFS思路

| 特性           | DFS（深度优先）          | BFS（广度优先）              |
| -------------- | ------------------------ | ---------------------------- |
| **数据结构**   | 系统栈（递归）或显式栈   | 队列 `queue`                 |
| **搜索顺序**   | **一条路走到黑**，再回溯 | **层层扩散**，像水波纹       |
| **空间复杂度** | O(递归深度) 最坏 O(m×n)  | O(队列大小) 最坏 O(min(m,n)) |
| **代码风格**   | 简洁，递归实现           | 稍长，显式维护队列           |
| **适用场景**   | 找路径、连通性           | 最短路径、层序遍历           |

# 主函数思路：与DFS相同

```c++
int numIslands(vector<vector<char>>& grid) {
        int n = grid.size(), m = grid[0].size();
        vector<vector<bool>> visited = vector<vector<bool>>(n, vector<bool>(m, false));

        int result = 0;
        for (int i = 0; i < n; i++) {
            for (int j = 0; j < m; j++) {
                if (!visited[i][j] && grid[i][j] == '1') {
                    result++; // 遇到没访问过的陆地，+1
                    bfs(grid, visited, i, j); // 将与其链接的陆地都标记上 true
                }
            }
        }
        return result;
    }
```

# BFS函数思路

- ## BFS使用队列`queue<pair<int, int>> que`，而不像DFS使用系统栈（递归）

  - ### 队列中储存当前层坐标

- ## BFS先将初始层入队并标记

  - ### 入队：`que.push({x, y});`

  - ### 标记：`visited[x][y] = true;`

- ## 当初始层不空的时候`while(!que.empty())`

  - ### 逐个取出当前层坐标

  ```cpp
   pair<int ,int> cur = que.front(); 
          que.pop();
          int curx = cur.first;
          int cury = cur.second;
  ```

  - ### 对其进行四个方向的移动：得到下一层的候选坐标

    - #### 边界检查

    - #### 符合条件的下一层坐标入队并标记

      - #### ` que.push({nextx, nexty});`

      - #### `visited[nextx][nexty] = true;`

  - ### 直到本层的坐标都空了，直接无缝衔接检查下一次

```c++
int dir[4][2] = {0, 1, 1, 0, -1, 0, 0, -1}; // 四个方向
void bfs(vector<vector<char>>& grid, vector<vector<bool>>& visited, int x, int y) {
    queue<pair<int, int>> que;//bfs核心数据结构：队列
    que.push({x, y});//初始层元素入队
    visited[x][y] = true; // 只要加入队列，立刻标记
    
    while(!que.empty()) //开始BFS扩散
    {
        //获取并弹出本层元素
        pair<int ,int> cur = que.front(); 
        que.pop();
        int curx = cur.first;
        int cury = cur.second;
        
        //对当前元素检查其下一层值
        for (int i = 0; i < 4; i++) 
        {
            int nextx = curx + dir[i][0];
            int nexty = cury + dir[i][1];
            if (nextx < 0 || nextx >= grid.size() || nexty < 0 || nexty >= grid[0].size()) continue;  // 越界了，直接跳过
            
            if (!visited[nextx][nexty] && grid[nextx][nexty] == '1') 
            {
                //满足条件的下一层值直接入队并标记
                que.push({nextx, nexty});
                visited[nextx][nexty] = true; // 只要加入队列立刻标记
            }
        }
    }
}
```

# 完整BFS代码

```c++
class Solution {
private:
    int dir[4][2]={0,1,1,0,-1,0,0,-1};
    void bfs(vector<vector<char>>& grid, vector<vector<bool>>& visited, int x, int y)
    {
        //初始化BFS队列
        queue<pair<int,int>> q;
        q.push({x,y});
        visited[x][y]=true;


        //进行BFS扩散
        while(!q.empty())
        {
            //获取并弹出当前队列值,得到当前坐标
            auto cur_pair=q.front();
            int curx=cur_pair.first;
            int cury=cur_pair.second;
            q.pop();

            //对当前坐标进行BFS扩散
            for(int i=0;i<4;i++)
            {
                int nextx=curx+dir[i][0];
                int nexty=cury+dir[i][1];

                //边界检查
                if(nextx<0||nextx>=grid.size()||nexty<0||nexty>=grid[0].size())
                {
                    continue;
                }
                //若符合条件：入队+标记
                if(grid[nextx][nexty]=='1'&&visited[nextx][nexty]==false)
                {
                    visited[nextx][nexty]=true;
                    q.push({nextx,nexty});
                }
            }

        }
        
        
    }
public:
    int numIslands(vector<vector<char>>& grid)
    {
        int n=grid.size(),m=grid[0].size();
        vector<vector<bool>> visited = vector<vector<bool>>(n,vector<bool>(m,false));
        int result=0;
        for(int i=0;i<n;i++)
        {
            for(int j=0;j<m;j++)
            {
                if(grid[i][j]=='1'&&!visited[i][j])
                {
                    visited[i][j]=true;
                    result++;
                    //调用BFS
                    bfs(grid, visited,i,j);
                }
            }
        }
        return result;
    }
};
```

