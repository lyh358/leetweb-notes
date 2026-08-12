# 力扣 543：二叉树的直径学习笔记

## 一、题目核心解析

**题目描述**：给定一棵二叉树，计算它的直径长度。二叉树的直径是指树中任意两个节点之间最长路径的长度。这条路径可能穿过根节点，也可能不穿过。

**核心定义**：路径长度以**边数**表示。例如，节点 A 到节点 B 经过 3 条边，路径长度即为 3。

**关键示例**：

对于二叉树，其结构如下：

```
      1
     / \
    2   3
   / \
  4   5
```

最长路径为 `4 → 2 → 1 → 3` 或 `5 → 2 → 1 → 3`，共经过 3 条边，因此直径为 3。

---
```
class Solution {
    //求直径本质上就是使用二叉树求深度的方法
public:
    int diameter = 0;

    int depth(TreeNode* root)
    {
        if(!root) return 0;

        int left=depth(root->left);     //求左侧深度
        int right=depth(root->right);   //求右侧深度

        diameter=max(diameter,left+right);  //全局直径更新：直径=左边长之和+右边长之和

        return 1+max(left,right);   //求深度返回条件：叶子节点是1+上面最长的路=当前深度
    }
    int diameterOfBinaryTree(TreeNode* root) {
        depth(root);
        return diameter;
    }
};
```
