# 102.二叉树的层序遍历

给你二叉树的根节点 `root` ，返回其节点值的 **层序遍历** 。 （即逐层地，从左到右访问所有节点）。

**示例 1：**

![img](https://assets.leetcode.com/uploads/2021/02/19/tree1.jpg)

```lua
输入：root = [3,9,20,null,null,15,7]
输出：[[3],[9,20],[15,7]]
```

**示例 2：**

```lua
输入：root = [1]
输出：[[1]]
```

**示例 3：**

```undefined
输入：root = []
输出：[]
```

**提示：**

- 树中节点数目在范围 `[0, 2000]` 内
- `-1000 <= Node.val <= 1000`

---

# 核心方法：二叉树的数据结构、BFS

```cpp
class Solution {
public:
    vector<vector<int>> levelOrder(TreeNode* root) {
        //层序遍历：简单的单层BFS
        queue<TreeNode*> q;
        vector<vector<int>> ans;
        //防空树！！！
        if(!root) return ans;

        //BFS之初始化
        q.push(root);
        //单层BFS：一层while+一层for
        while(!q.empty()) 
        {
            int size = q.size();
            vector<int> level;
            for(int i=0;i<size;i++)
            {
                TreeNode* temp = q.front();
                q.pop();
                level.push_back(temp->val);
                if(temp->left) q.push(temp->left);
                if(temp->right) q.push(temp->right);
            }
            ans.push_back(level);
        }
        return ans;
    }
};
```

---
