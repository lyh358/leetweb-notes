# 🌲 C++ 二叉树刷题基础笔记（LeetCode 100 版）

这份笔记聚焦**刷题实战**，从概念到代码模板，帮你快速上手。

---

## 一、二叉树基础概念

### 1. 什么是二叉树

二叉树（Binary Tree）是每个节点**最多有两个子节点**的树结构，分别称为**左子树**和**右子树**。

```
        1          ← 根节点 (root)
       /       2   3        ← 内部节点
     / \   \
    4   5   6      ← 叶子节点 (leaf)
```

### 2. 核心术语

| 术语 | 含义 |
|------|------|
| **根节点** | 最顶部的节点，没有父节点 |
| **叶子节点** | 没有子节点的节点 |
| **深度** | 从根到该节点的边数（根深度为 0 或 1，LeetCode 通常按 1 算） |
| **高度** | 从该节点到最远叶子的边数（叶子高度为 0 或 1） |
| **层** | 同一深度的所有节点 |

### 3. 常见二叉树类型

| 类型 | 定义 | 特点 |
|------|------|------|
| **满二叉树** | 每个非叶子节点都有 2 个子节点 | 第 `k` 层有 `2^(k-1)` 个节点 |
| **完全二叉树** | 除最后一层外全满，最后一层靠左对齐 | 可用数组存储，适合堆结构 |
| **二叉搜索树 BST** | 左子树所有值 < 根 < 右子树所有值 | 中序遍历结果为升序 |
| **平衡二叉树** | 任意节点左右子树高度差 ≤ 1 | AVL、红黑树等，保证 O(log n) 操作 |

### 4. 二叉树的基本性质

- 第 `i` 层最多有 `2^(i-1)` 个节点
- 高度为 `h` 的二叉树最多有 `2^h - 1` 个节点
- 对于任意二叉树：**叶子节点数 = 度为 2 的节点数 + 1**
- `n` 个节点的完全二叉树高度：`⌊log₂n⌋ + 1`

### 5. 存储方式

**链式存储**（LeetCode 标准，最常用）：
```cpp
struct TreeNode {
    int val;
    TreeNode *left;
    TreeNode *right;
};
```

**数组存储**（完全二叉树适用）：
- 节点 `i` 的左子节点在 `2*i + 1`
- 节点 `i` 的右子节点在 `2*i + 2`
- 节点 `i` 的父节点在 `(i-1)/2`

---

## 二、二叉树节点定义（LeetCode 标准版）

```cpp
struct TreeNode {
    int val;
    TreeNode *left;
    TreeNode *right;
    TreeNode() : val(0), left(nullptr), right(nullptr) {}
    TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
    TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
};
```

---

## 三、四大遍历（递归 + 迭代）

### 1. 前序遍历（根 → 左 → 右）

**递归版**
```cpp
void preorder(TreeNode* root, vector<int>& res) {
    if (!root) return;
    res.push_back(root->val);      // 访问根
    preorder(root->left, res);     // 左
    preorder(root->right, res);    // 右
}
```

**迭代版（栈）**
```cpp
vector<int> preorderTraversal(TreeNode* root) {
    vector<int> res;
    stack<TreeNode*> st;
    if (root) st.push(root);
    while (!st.empty()) {
        TreeNode* node = st.top(); st.pop();
        res.push_back(node->val);
        if (node->right) st.push(node->right);  // 右先入栈，后出
        if (node->left)  st.push(node->left);
    }
    return res;
}
```

---

### 2. 中序遍历（左 → 根 → 右）

**递归版**
```cpp
void inorder(TreeNode* root, vector<int>& res) {
    if (!root) return;
    inorder(root->left, res);
    res.push_back(root->val);
    inorder(root->right, res);
}
```

**迭代版（最常用！）**
```cpp
vector<int> inorderTraversal(TreeNode* root) {
    vector<int> res;
    stack<TreeNode*> st;
    TreeNode* cur = root;
    while (cur || !st.empty()) {
        if (cur) {
            st.push(cur);
            cur = cur->left;      // 一路向左
        } else {
            cur = st.top(); st.pop();
            res.push_back(cur->val);  // 访问根
            cur = cur->right;         // 转向右
        }
    }
    return res;
}
```

---

### 3. 后序遍历（左 → 右 → 根）

**递归版**
```cpp
void postorder(TreeNode* root, vector<int>& res) {
    if (!root) return;
    postorder(root->left, res);
    postorder(root->right, res);
    res.push_back(root->val);
}
```

**迭代版（根右左，再反转 = 左右根）**
```cpp
vector<int> postorderTraversal(TreeNode* root) {
    vector<int> res;
    stack<TreeNode*> st;
    if (root) st.push(root);
    while (!st.empty()) {
        TreeNode* node = st.top(); st.pop();
        res.push_back(node->val);
        if (node->left)  st.push(node->left);
        if (node->right) st.push(node->right);
    }
    reverse(res.begin(), res.end());
    return res;
}
```

---

### 4. 层序遍历（BFS，必会！）

```cpp
vector<vector<int>> levelOrder(TreeNode* root) {
    vector<vector<int>> res;
    if (!root) return res;
    queue<TreeNode*> q;
    q.push(root);
    while (!q.empty()) {
        int size = q.size();          // 当前层节点数
        vector<int> level;
        for (int i = 0; i < size; i++) {
            TreeNode* node = q.front(); q.pop();
            level.push_back(node->val);
            if (node->left)  q.push(node->left);
            if (node->right) q.push(node->right);
        }
        res.push_back(level);
    }
    return res;
}
```

---

## 四、递归解题万能框架

> **99% 的二叉树递归题都可以套这个模板**

```cpp
// 定义：输入根节点，返回我要的信息
ReturnType dfs(TreeNode* root) {
    // 1. 空节点 base case
    if (!root) return ...;

    // 2. 向左右子树要信息（后序位置）
    ReturnType left  = dfs(root->left);
    ReturnType right = dfs(root->right);

    // 3. 根据左右子树信息 + 当前节点，整合出当前层答案
    ReturnType res = 处理(left, right, root);

    return res;
}
```

**关键思想**：不要想递归怎么一层层展开，而是把 `dfs(root)` 当成一个黑盒函数——传入根节点，它就能返回正确答案。

---

## 五、高频基础函数（直接背）

### 1. 求树高（深度）
```cpp
int maxDepth(TreeNode* root) {
    if (!root) return 0;
    return 1 + max(maxDepth(root->left), maxDepth(root->right));
}
```

### 2. 求节点个数
```cpp
int countNodes(TreeNode* root) {
    if (!root) return 0;
    return 1 + countNodes(root->left) + countNodes(root->right);
}
```

### 3. 判断平衡二叉树
```cpp
// 返回高度，-1 表示不平衡
int dfs(TreeNode* root) {
    if (!root) return 0;
    int left = dfs(root->left);
    if (left == -1) return -1;
    int right = dfs(root->right);
    if (right == -1) return -1;
    if (abs(left - right) > 1) return -1;
    return 1 + max(left, right);
}
bool isBalanced(TreeNode* root) { return dfs(root) != -1; }
```

### 4. 判断对称二叉树
```cpp
bool check(TreeNode* l, TreeNode* r) {
    if (!l && !r) return true;
    if (!l || !r) return false;
    return l->val == r->val 
        && check(l->left, r->right) 
        && check(l->right, r->left);
}
bool isSymmetric(TreeNode* root) {
    return !root || check(root->left, root->right);
}
```

---

## 六、二叉搜索树（BST）核心性质

> **左 < 根 < 右**，中序遍历结果是有序数组！

### 1. 验证 BST
```cpp
// 每个节点有上下界
bool isValidBST(TreeNode* root, long long low = LLONG_MIN, long long high = LLONG_MAX) {
    if (!root) return true;
    if (root->val <= low || root->val >= high) return false;
    return isValidBST(root->left, low, root->val) 
        && isValidBST(root->right, root->val, high);
}
```

### 2. BST 中序遍历 = 升序数组
```cpp
// 第 K 小元素：中序遍历到第 K 个即可
int kthSmallest(TreeNode* root, int k) {
    stack<TreeNode*> st;
    TreeNode* cur = root;
    while (cur || !st.empty()) {
        while (cur) { st.push(cur); cur = cur->left; }
        cur = st.top(); st.pop();
        if (--k == 0) return cur->val;
        cur = cur->right;
    }
    return -1;
}
```

---

## 七、常见题型分类 & 技巧

| 题型 | 代表题 | 技巧 |
|------|--------|------|
| **遍历基础** | 144/94/145/102 | 背熟四种遍历模板 |
| **属性计算** | 104/111/222/110 | 递归返回 int（高度/深度/节点数） |
| **路径问题** | 112/113/124/437 | 回溯 / 前缀和 |
| **构造二叉树** | 105/106/889 | 找根节点 + 递归构造左右子树 |
| **BST 专项** | 98/700/701/450 | 利用 BST 性质，中序有序 |
| **公共祖先** | 236/235 | 后序遍历，p/q 分居两侧即为答案 |
| **序列化** | 297/449 | 前序遍历 + 特殊标记空节点 |

---

## 八、易错点提醒

1. **`if (!root)` 和 `if (root == nullptr)`** 等价，用前者更简洁
2. **递归不要想太深**，相信子调用返回正确结果
3. **层序遍历**中 `int size = q.size()` 必须在 `for` 之前固定，因为 `q.size()` 在循环中会变化
4. **BST 验证**不能只看左右子节点，要维护上下界
5. **修改树结构**的题目（如翻转、删除），注意返回新的根节点

---

## 九、推荐刷题顺序（由易到难）

```
1. 144/94/145/102 → 遍历热身
2. 104/111/110/222 → 属性计算
3. 101/226/112/257 → 简单递归
4. 236/235/98/700 → 经典套路
5. 105/106/437/124 → 进阶综合
```

祝你刷题顺利！🚀
