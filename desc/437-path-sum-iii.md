# 📘 LeetCode 437 — 路径总和 III（DFS 解法详解）

---

## 一、题目信息

| 项目 | 内容 |
|------|------|
| 题号 | 437 |
| 难度 | 中等 |
| 标签 | 树、深度优先搜索、二叉树 |
| 核心思路 | 双重 DFS：枚举起点 + 向下搜索 |

**题目描述**：给定一个二叉树的根节点 `root` 和一个整数 `targetSum`，求**路径和等于 `targetSum` 的路径数量**。

**路径要求**：
- 不需要从根节点开始
- 不需要在叶子节点结束
- 必须是**向下的**（只能从父节点到子节点）

```
输入：root = [10,5,-3,3,2,null,11,3,-2,null,1], targetSum = 8

        10
       /  \
      5   -3
     / \    \
    3   2   11
   / \   \
  3  -2   1

和为 8 的路径有 3 条：
① 5 → 3（5+3=8）
② 5 → 2 → 1（5+2+1=8）
③ -3 → 11（-3+11=8）
```

---
官网上有bug，需要把sy
```
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     long long val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(long long x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(long long x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    //内层DFS：得到以root为起点的路径和
    long long countPath(TreeNode* root, long long targetSum)
    {
        if(!root) return 0;

        long long count = 0;

        if(root->val == targetSum) count++;

        count += countPath(root->left, targetSum - root->val);
        count += countPath(root->right, targetSum - root->val);

        return count;
    }



    //外层DFS，遍历所有的节点，统计以他们为起点的路径之和
    long long pathSum(TreeNode* root, long long targetSum) {
        if(!root) return 0;
        return countPath(root,targetSum) + pathSum(root->left,targetSum) + pathSum(root->right,targetSum);
    }
};
```
