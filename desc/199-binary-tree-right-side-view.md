# 📘 LeetCode 199 — 二叉树的右视图（学习笔记）

---

## 一、题目信息

| 项目 | 内容 |
|------|------|
| 题号 | 199 |
| 难度 | 中等 |
| 标签 | 树、深度优先搜索、广度优先搜索、二叉树 |
| 核心思路 | BFS 层序遍历，取每层最后一个节点 |

**题目描述**：给定一棵二叉树，想象自己站在它的右侧，按照从顶部到底部的顺序，返回从右侧所能看到的节点值。

```
    1            ← 能看到 1
   / \
  2   3          ← 能看到 3
   \   \
    5   4        ← 能看到 4

从右侧看：[1, 3, 4]
```

---
```
class Solution {
public:
    vector<int> rightSideView(TreeNode* root) {
        vector<int> ans;
        if(!root) return ans; 

        queue<TreeNode*> q;
        q.push(root);

        while(!q.empty())
        {
            int levelsize = q.size();
            vector<int> level;

            for(int i=0;i<levelsize;i++)
            {
                TreeNode* temp = q.front();
                q.pop();
                level.push_back(temp->val);

                if(temp->left) q.push(temp->left);
                if(temp->right) q.push(temp->right);
            }
            ans.push_back(level.back());
        }
        return ans;
    }
};
```
