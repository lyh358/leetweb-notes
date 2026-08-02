## 核心算法：DFS（深度优先搜索）、递归

## 核心数据结构：

### 1.二维动态数组`vector<vector<int>> result, graph;`

- #### `result`用于存放可行的一维路径；

- #### `graph`用于输入无向图；

### 2.一维动态数组`vector<int> path;`

- #### 用于存放0节点到终点的路径

---

## 核心思路part1：

#### 先定义二维动态数组result和一维动态数组path

```c++
    vector<vector<int>> result; // 收集符合条件的路径
    vector<int> path; // 0节点到终点的路径
```

#### 编写DFS函数逻辑

##### dfs()函数：

- **返回值：void**
- **参数1：二维数组graph的引用**
- **参数2：int x，代表当前正在遍历的节点**

#### dfs函数包括：

- **终止条件：**当前节点到达终点`if(x==graph.size()-1)`
  - 把当前path加入result并退出
- **循环遍历当前节点的子节点：**` for(int i=0; i<graph[x].size(); i++)`
  - **把当前子节点（graph[x] [i]）加入path**：`path.push_back(graph[x][i]);`
  - **递归下一层**`dfs(graph,graph[x][i])`
  - **回溯**：撤销本节点`path.pop_back();`

```c++
	void dfs(vector<vector<int>>& graph,int x)
    {
        //输出条件是：从节点 0 到节点 n-1 的路径
        //也就是x遍历到了graph.size() - 1
        if(x==graph.size()-1)
        {
            //把可行的路径储存到二维动态数组result中
            result.push_back(path);
            //返回
            return;
        }
        
       //for循环遍历当前节点的所有子节点graph[x].size()
        for(int i=0; i<graph[x].size(); i++)// 遍历节点n链接的所有节点
        {
            path.push_back(graph[x][i]);//把遍历到的节点加入路径
            dfs(graph,graph[x][i]);//进入下一层递归
            path.pop_back();//回溯，撤销本节点
        }
    }
```

## 核心思路part2：执行DFS

### `allPathsSourceTarget`函数：

- ### 传入有向无环图

- ### 初始化初始节点x=0

  - **将0加入path**`path.push_back(0);`
  - **调用dfs函数**`dfs(graph, 0);`

- ### 返回二维动态数组result`return result;`

```c++
 vector<vector<int>> allPathsSourceTarget(vector<vector<int>>& graph) 
 {
        path.push_back(0); // 无论什么路径已经是从0节点出发
        dfs(graph, 0); // 开始遍历
        return result;
    }
```

---



## 完整代码

```c++
class Solution {
private:
    vector<vector<int>> result;
    vector<int> path;

    void dfs(vector<vector<int>>& graph, int x)
    {
        if(x==graph.size()-1)
        {
            result.push_back(path);
            return;
        }
        for(int i = 0;i<graph[x].size();i++)
        {
            path.push_back(graph[x][i]);
            dfs(graph, graph[x][i]);
            path.pop_back();
        }
    }
public:
    vector<vector<int>> allPathsSourceTarget(vector<vector<int>>& graph) 
    {
        path.push_back(0);
        dfs(graph, 0);
        return result;
    }
};
```

