# 📘 LeetCode 230 — 二叉搜索树中第K小的元素（学习笔记）

---

## 一、题目信息

| 项目 | 内容 |
|------|------|
| 题号 | 230 |
| 难度 | 中等 |
| 标签 | 树、深度优先搜索、二叉搜索树、二叉树 |
| 核心思路 | 中序遍历 + 计数器 |

---
```
class Solution {
public:
    //全局计数器和答案
    int count=0;
    int ans=0;

    //中序遍历并计数
    void inorder(TreeNode* root,int k)
    {
        //返回条件
        if(!root) return;
        //遍历左
        inorder(root->left,k);
        //遍历中：当前节点，计数++，判断是否输出
        count++;
        if(count==k)
        {
            ans = root->val;
        }
        //遍历右
        inorder(root->right,k);
    }

    int kthSmallest(TreeNode* root, int k) {
        inorder(root,k);
        return ans;
    }
};
