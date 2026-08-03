# 力扣 226：翻转二叉树学习笔记

## 一、题目核心解析

- **题目描述**：给你一棵二叉树的根节点 `root`，翻转这棵二叉树，并返回其根节点。
- **核心定义**：翻转二叉树，本质上就是生成原树的镜像。即树中的每一个节点，都需要交换它的左孩子和右孩子。
- **关键示例**：给出翻转前后的二叉树结构图（用文本形式表示）：

翻转前：

```text
      4
     / \
    2   7
   / \ / \
  1  3 6  9
```

翻转后：

```text
      4
     / \
    7   2
   / \ / \
  9  6 3  1
```

## 二、核心思想：如何透彻理解递归？

- 不要试图在脑海中模拟整棵树的翻转过程。递归的本质就是"把同一件事交给每个节点去做"。
- 你只需要问自己：如果当前只有一个节点，我该做什么？答案：交换它的左右孩子。
- **递归三步法**：
  1. **确定递归出口（边界条件）**：如果当前节点为空（`root == nullptr`），说明已经走到叶子节点之外，无需翻转，直接返回 `nullptr`。
  2. **单层递归逻辑**：分别递归翻转当前节点的左子树和右子树。
  3. **结果合并（核心操作）**：将翻转后的右子树赋给当前节点的左孩子，将翻转后的左子树赋给当前节点的右孩子，最后返回当前节点。
- **顿悟时刻**：你不需要关心"怎么把整棵左子树搬到右边"，只要你在每个节点上执行一次"交换左右孩子"的操作，递归会自动帮你把整棵树剩下的事全部做完。

## 三、完整代码实现 (C++)

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
    TreeNode* invertTree(TreeNode* root) {
        // 1. 确定递归出口（边界条件）
        // 如果当前节点为空，直接返回 nullptr
        if (root == nullptr) {
            return nullptr;
        }

        // 2. 单层递归逻辑 & 3. 结果合并
        // 递归翻转左子树和右子树，并直接将返回值赋给对方的指针
        // 此时我们坚信递归函数能完美翻转子树并返回新的根节点
        TreeNode* temp = root->left;      // 暂存左孩子，防止被覆盖
        root->left = invertTree(root->right);  // 翻转右子树并赋给左孩子
        root->right = invertTree(temp);        // 翻转原左子树并赋给右孩子

        // 返回当前处理完的节点
        return root;
    }
};
```

## 四、复杂度分析

- **时间复杂度**：O(n)，其中 n 是二叉树的节点数量。每个节点恰好被访问一次，执行常数时间的交换操作。
- **空间复杂度**：O(h)，其中 h 为树的高度，主要消耗在递归调用栈上。
  - 最好情况（完全平衡二叉树）：O(log n)
  - 最坏情况（退化为链表）：O(n)

## 五、拓展思路：迭代法（层序遍历 BFS）

- 除了递归，我们也可以使用队列模拟层序遍历。逐层遍历二叉树，遇到非空节点就交换它的左右孩子，然后将非空的左右孩子入队。
- **完整的 C++ 参考代码**：

```cpp
class Solution {
public:
    TreeNode* invertTree(TreeNode* root) {
        if (root == nullptr) return nullptr;

        std::queue<TreeNode*> q;
        q.push(root);

        while (!q.empty()) {
            TreeNode* node = q.front();
            q.pop();

            // 交换当前节点的左右孩子
            TreeNode* temp = node->left;
            node->left = node->right;
            node->right = temp;

            // 将非空的子节点入队，继续处理下一层
            if (node->left) q.push(node->left);
            if (node->right) q.push(node->right);
        }

        return root;
    }
};
```

- **BFS 复杂度分析**：
  - **时间复杂度**：O(n)，同样需要遍历所有节点一次。
  - **空间复杂度**：O(n)，队列中最多同时存储树的最大宽度个节点。在完全二叉树中，底层节点数约为 n/2，因此空间复杂度为 O(n)。相比递归的 O(h)，BFS 在极端宽树情况下内存占用更大，但在极端深树（链表）情况下不会发生栈溢出。

## 六、学习总结

1. **打破思维定势**：翻转二叉树不是机械地按层级位置交换，而是每个节点独立执行"交换左右孩子"的动作。
2. **递归的精髓**：相信你的递归函数。当你写下 `invertTree(root->left)` 时，你要坚信它已经完美地翻转了左子树并返回了新的根节点。
3. **DFS vs BFS 对比**：DFS（递归）代码极简，逻辑清晰，面试首选；BFS（迭代）利用队列模拟，内存使用可控，适合处理深度极大但宽度有限的树，可有效避免递归导致的栈溢出问题。
