# 📘 LeetCode 114 — 二叉树展开为链表（双解法详解）

---

## 一、题目信息

| 项目 | 内容 |
|------|------|
| 题号 | 114 |
| 难度 | 中等 |
| 标签 | 栈、树、深度优先搜索、链表、二叉树 |
| 核心思路 | 后序遍历递归 / 寻找前驱节点迭代 |

**题目要求**：给定一棵二叉树，将其**原地**展开为一个"链表"：
- 展开后的顺序是二叉树的**前序遍历**顺序
- 所有节点的 `left` 指针都置为 `nullptr`
- 所有节点的 `right` 指针指向下一个节点
- 不能创建新节点，只能调整指针

```
输入：
    1
   / \
  2   5
 / \   \
3   4   6

前序遍历：1 → 2 → 3 → 4 → 5 → 6

输出（链表形式）：
1
   2
       3
           4
               5
                   6

即：1->right=2, 2->right=3, 3->right=4, 4->right=5, 5->right=6
    所有 left = nullptr
```

---
采用右左中的

```
class Solution {
public:
    TreeNode* prevNode = nullptr;
    void flatten(TreeNode* root) {
        if(!root) return;

        flatten(root->right);
        flatten(root->left);

        root->left=nullptr;
        root->right=prevNode;
        prevNode=root;
    }
};
```
