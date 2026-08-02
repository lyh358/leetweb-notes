# 102.二叉树的层序遍历

给你二叉树的根节点 `root` ，返回其节点值的 **层序遍历** 。 （即逐层地，从左到右访问所有节点）。

**示例 1：**

![img](https://assets.leetcode.com/uploads/2021/02/19/tree1.jpg)

```
输入：root = [3,9,20,null,null,15,7]
输出：[[3],[9,20],[15,7]]
```

**示例 2：**

```
输入：root = [1]
输出：[[1]]
```

**示例 3：**

```
输入：root = []
输出：[]
```

**提示：**

- 树中节点数目在范围 `[0, 2000]` 内
- `-1000 <= Node.val <= 1000`

---

# 核心方法：二叉树的数据结构、BFS

---

# 核心思路

## （1）创建二维结果数组`result`

```c++
// 结果数组：外层vector存储层，内层vector存储该层的节点值
// 最终形如 [[3], [9,20], [15,7]]
vector<vector<int>> result;
```

## （2）处理空树，避免访问空指针

- ### 若为空树，直接返回空结果数组result

```c++
if(!root)
{
    return result;
}
```

## （3）创建BFS的核心数据结构：队列queue

- ### 队列中存放的是二叉树的节点`<TreeNode*>`

```c++
queue<TreeNode*> q;
```

- ### 将根节点入队

  - #### 作为第一层待搜索节点

```c++
queue.push(root);
```

## （3）开始广度优先搜索

- ### 当队列q不为空时，说明还有子节点未被搜索，则继续

```c++
while (!q.empty())
 {
     
 }
```

- ### 记录当前层的节点数

```cpp
int levelSize = q.size();
```

- ### 建立一个一维临时数组`currentlevel`储存当前层的节点值

  - #### 每轮while循环都会新建一个`currentlevel`，代表新的一层

```cpp
vector<int> currentLevel;
```

- ### 处理当前层的所有节点

  - #### 依次从队列q中获取并弹出当前层所有节点

    - ##### `TreeNode* node = q.front();`

    - ##### `q.pop();`

  - #### 将节点的值存入`currentLevel`数组

    - ##### `currentLevel.push_back(node->val);`

  - #### 将当前层每个节点的左右子节点入队（如果有），但是先不进行处理（获取值），下一轮再进行处理

    - ##### `if (node->left)  q.push(node->left);`

    - ##### `if (node->right) q.push(node->right);`

```cpp
 		   // 【关键】只处理当前层的节点（levelSize个）
            // 在循环内部我们会入队下一层节点，但由于i < levelSize，不会处理它们
            for (int i = 0; i < levelSize; i++) {
                
                // 取出队首节点（最先入队的当前层节点）
                TreeNode* node = q.front();//队列是FIFO的，所以要用q.front()而不是q.back();
                
                // 出队（已访问完毕）
                q.pop();
                
                // 记录该节点值到当前层
                currentLevel.push_back(node->val);
                
                // 将左右孩子入队（这些是下一层的节点）
                // 注意：它们入队后，本轮for循环不会处理（因为i只到levelSize-1）
                if (node->left)  q.push(node->left);
                if (node->right) q.push(node->right);
            }
```

- ### 当前层处理完毕，将currentLevel加入最终结果

  - #### 此时队列中恰好只有下一层的所有节点（levelSize个）

  - #### 下一轮while循环会处理它们

```cpp
result.push_back(currentLevel);
```

## （4）返回结果

- ### 所有层处理完毕，返回二维数组`result`

```cpp
 return result;
```

---

# 完整代码

```cpp
class Solution {
public:
    vector<vector<int>> levelOrder(TreeNode* root) {
        // 结果数组：外层vector存储层，内层vector存储该层的节点值
        // 最终形如 [[3], [9,20], [15,7]]
        vector<vector<int>> result;
        
        // 处理空树：如果根为空，直接返回空数组（避免后面访问空指针）
        if (!root) return result;
        
        // 声明FIFO队列，存储TreeNode指针
        // 队列是实现BFS的核心数据结构：先访问的节点，其孩子也要先访问
        queue<TreeNode*> q;
        
        // 根节点入队，作为BFS的起点
        q.push(root);
        
        // 当队列不为空时继续循环（还有节点待处理）
        while (!q.empty()) {
            
            // 【核心技巧】记录当前层的节点数量
            // 此时队列中恰好只有当前层的所有节点（上一层处理完，下一层还未入队）
            int levelSize = q.size();
            
            // 临时数组，存储当前层的所有节点值
            // 每轮while循环都会新建一个，代表新的一层
            vector<int> currentLevel;
            
            // 【关键】只处理当前层的节点（levelSize个）
            // 在循环内部我们会入队下一层节点，但由于i < levelSize，不会处理它们
            for (int i = 0; i < levelSize; i++) {
                
                // 取出队首节点（最先入队的当前层节点）
                TreeNode* node = q.front();
                
                // 出队（已访问完毕）
                q.pop();
                
                // 记录该节点值到当前层
                currentLevel.push_back(node->val);
                
                // 将左右孩子入队（这些是下一层的节点）
                // 注意：它们入队后，本轮for循环不会处理（因为i只到levelSize-1）
                if (node->left)  q.push(node->left);
                if (node->right) q.push(node->right);
            }
            
            // 当前层处理完毕，将currentLevel加入最终结果
            result.push_back(currentLevel);
            
            // 此时队列中恰好只有下一层的所有节点（levelSize个）
            // 下一轮while循环会处理它们...
        }
        
        // 所有层处理完毕，返回二维数组
        return result;
    }
};
```