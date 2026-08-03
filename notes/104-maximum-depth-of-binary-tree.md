# 力扣 104：二叉树的最大深度学习笔记

## 一、 题目核心解析

**题目描述**：给定一个二叉树 `root`，返回其最大深度。

**核心定义**：二叉树的最大深度是指从根节点到最远叶子节点的最长路径上的节点数。

**关键示例**：

对于二叉树 `[3,9,20,null,null,15,7]`，其结构如下：

```text
      3
     / \
    9  20
       / \
      15  7
```

最长路径为 `3 → 20 → 15`（或 `3 → 20 → 7`），包含 3 个节点，因此最大深度为 3。

---

## 二、 核心思想：递归分治

二叉树天然适合用递归解决。面对树形问题，不要一开始就陷入细节，而是要思考**整棵树与其左右子树的关系**。

**核心逻辑公式**：

> 一棵二叉树的最大深度 = 1（当前根节点） + max(左子树最大深度, 右子树最大深度)

**递归三步法**：

1. **确定递归出口（边界条件）**：当遍历到空节点（`root == nullptr`）时，说明该分支走到尽头，深度为 0，直接返回。
2. **单层递归逻辑**：分别递归求出左子树的深度和右子树的深度。
3. **结果合并**：取左右子树深度中较大的值，加上当前节点所在的 1 层，作为当前树的深度向上返回。

---

## 三、 完整代码实现 (C++)

```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    int maxDepth(TreeNode* root) {
        // 1. 递归出口：空节点深度为 0
        if (root == nullptr) {
            return 0;
        }

        // 2. 递归计算左右子树的最大深度
        int leftDepth = maxDepth(root->left);
        int rightDepth = maxDepth(root->right);

        // 3. 取左右子树深度的最大值，加上当前节点层数 1
        return max(leftDepth, rightDepth) + 1;
    }
};
```

---

## 四、 复杂度分析

- **时间复杂度：O(n)**

  其中 n 为二叉树的节点总数。在递归过程中，每个节点都恰好被访问一次。

- **空间复杂度：O(h)**

  其中 h 为二叉树的高度。空间消耗主要来自于递归调用栈的深度，它等于树的高度。

  - **最好情况**：树完全平衡，高度 h = log n，空间复杂度为 O(log n)。
  - **最坏情况**：树退化为单边链表（如只有右子树），高度 h = n，空间复杂度为 O(n)。

---

## 五、 拓展思路：迭代法（层序遍历 BFS）

如果不想使用递归，或者担心树过深导致栈溢出，可以使用广度优先搜索（BFS）逐层遍历。

**核心思路**：

使用队列（Queue）逐层遍历，每处理完一层的所有节点，深度就 +1，直到队列为空。

**参考代码**：

```cpp
class Solution {
public:
    int maxDepth(TreeNode* root) {
        if (root == nullptr) return 0;

        queue<TreeNode*> q;
        q.push(root);
        int depth = 0;

        while (!q.empty()) {
            int size = q.size(); // 记录当前层的节点数量
            // 遍历当前层的所有节点
            for (int i = 0; i < size; i++) {
                TreeNode* node = q.front();
                q.pop();
                // 将下一层的节点加入队列
                if (node->left) q.push(node->left);
                if (node->right) q.push(node->right);
            }
            // 当前层遍历完毕，深度 +1
            depth++;
        }
        return depth;
    }
};
```

**BFS 复杂度对比**：

- 时间复杂度依然是 O(n)。
- 空间复杂度为 O(n)，因为队列在最坏情况下（完全二叉树的最后一层）需要容纳约 n/2 个节点。

---

## 六、 学习总结

1. **树的递归本质**：将大问题拆解为"根节点 + 左子树 + 右子树"的子问题。
2. **DFS vs BFS**：
   - **DFS（递归）**：代码极简，书写最快，逻辑最直观，面试首选。
   - **BFS（迭代）**：适合逐层处理问题，可作为避免递归栈溢出的替代方案。
3. **避坑指南**：在写树的递归时，务必先想清楚**返回值代表什么**（本题中代表以当前节点为根的树的最大深度），以及**基准情况（空节点）返回什么**。
