# LeetCode 98 - 验证二叉搜索树（学习笔记）

## 一、题目信息

| 项目 | 内容 |
|------|------|
| 题号 | 98 |
| 难度 | 中等 |
| 标签 | 树、深度优先搜索、二叉搜索树、二叉树 |
| 核心思路 | 中序遍历 + 前驱节点比较 |

---

```
class Solution {
public:
    //二叉搜索时BST一定按中序遍历递增
    long long prev = LLONG_MIN;
    bool isValidBST(TreeNode* root) {
        if(!root) return true;   //空树属于BST


        //中序遍历
        //先检查左子树是不是BST
        if(!isValidBST(root->left)) return false;

        //核心操作：检查当前节点是不是不小于之前的节点
        if(root->val <=prev) return false;
        //更新prev值
        prev = root->val;

        //检查右子树
        if(!isValidBST(root->right)) return false;

        return true;
    }
};
```