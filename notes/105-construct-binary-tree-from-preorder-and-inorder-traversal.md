# 📘 力扣105：从前序与中序遍历序列构造二叉树 — C++ 学习笔记

> **题目链接**：[LeetCode 105. Construct Binary Tree from Preorder and Inorder Traversal](https://leetcode.com/problems/construct-binary-tree-from-preorder-and-inorder-traversal/)  
> **难度**：中等  
> **核心方法**：递归分治 + 哈希表快速定位

---

## 一、核心思路（两张图秒懂）

### 🔑 遍历特性

| 遍历方式 | 顺序 | 关键特性 |
|---------|------|---------|
| **前序遍历** | 根 → 左子树 → 右子树 | **第一个元素永远是当前子树的根** |
| **中序遍历** | 左子树 → 根 → 右子树 | **根左边全是左子树，根右边全是右子树** |

### 🔑 解题三步走

```
① 在前序数组中找到根节点（第一个元素）
② 在中序数组中找到根的位置，划分出左、右子树
③ 递归对左、右子树重复 ①②
```

---

## 二、区间划分图解（重点！）

结合题目官方图解，理解**前序**和**中序**数组的区间对应关系：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           前 序 遍 历                                    │
│  ┌────┬─────────────────────────────┬─────────────────────────────────┐  │
│  │ 根 │         左子树               │            右子树               │  │
│  └────┴─────────────────────────────┴─────────────────────────────────┘  │
│   ↑    ↑                             ↑                                 ↑ │
│ preLeft preLeft+1          preLeft+size_left_subtree      preLeft+size_left_subtree+1  preRight│
│                                                                      (= pIndex-inLeft+preLeft+1)│
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                           中 序 遍 历                                    │
│  ┌─────────────────────────────┬────┬─────────────────────────────────┐  │
│  │         左子树               │ 根 │            右子树               │  │
│  └─────────────────────────────┴────┴─────────────────────────────────┘  │
│   ↑                           ↑    ↑                                 ↑ │
│ inLeft              inorder_root-1 inorder_root              inorder_root+1  inRight│
│                              (= pIndex-1)  (= pIndex)                                    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 📐 关键公式

```cpp
// 左子树的节点个数 = 中序中根的位置 - 中序左边界
int size_left_subtree = inorder_root - inorder_left;

// 前序左子树区间：[preorder_left + 1,  preorder_left + size_left_subtree]
// 前序右子树区间：[preorder_left + size_left_subtree + 1,  preorder_right]
// 中序左子树区间：[inorder_left,  inorder_root - 1]
// 中序右子树区间：[inorder_root + 1,  inorder_right]
```

> 💡 **为什么前序左子树的右边界是 `preorder_left + size_left_subtree`？**  
> 因为前序中根后面紧跟着的就是**整个左子树**的前序遍历，左子树有 `size_left_subtree` 个节点，所以从 `preorder_left + 1` 开始，往后数 `size_left_subtree` 个就是左子树。

---

## 三、完整代码（带逐行注释）

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
private:
    // 哈希表：记录中序遍历中每个值对应的下标
    // 作用：O(1) 时间快速找到根节点在中序中的位置
    unordered_map<int, int> index;

public:
    /**
     * 递归构建二叉树
     * 
     * @param preorder       前序遍历数组
     * @param inorder        中序遍历数组
     * @param preorder_left  前序区间左边界（闭区间）
     * @param preorder_right 前序区间右边界（闭区间）
     * @param inorder_left   中序区间左边界（闭区间）
     * @param inorder_right  中序区间右边界（闭区间）
     */
    TreeNode* myBuildTree(const vector<int>& preorder, const vector<int>& inorder,
                          int preorder_left, int preorder_right,
                          int inorder_left, int inorder_right) {

        // ========== 递归终止条件 ==========
        // 如果前序左边界 > 右边界，说明当前子树没有节点，返回空
        if (preorder_left > preorder_right) {
            return nullptr;
        }

        // ========== Step 1: 前序第一个元素就是根节点 ==========
        int preorder_root = preorder_left;

        // ========== Step 2: 在中序中找到根的位置 ==========
        // 利用哈希表 O(1) 查找
        int inorder_root = index[preorder[preorder_root]];

        // ========== Step 3: 创建根节点 ==========
        TreeNode* root = new TreeNode(preorder[preorder_root]);

        // ========== Step 4: 计算左子树的节点个数 ==========
        // 中序中：根左边所有元素都是左子树的节点
        int size_left_subtree = inorder_root - inorder_left;

        // ========== Step 5: 递归构建左子树 ==========
        // 前序左子树区间：[preorder_left + 1, preorder_left + size_left_subtree]
        // 中序左子树区间：[inorder_left, inorder_root - 1]
        root->left = myBuildTree(preorder, inorder,
                                 preorder_left + 1,
                                 preorder_left + size_left_subtree,
                                 inorder_left,
                                 inorder_root - 1);

        // ========== Step 6: 递归构建右子树 ==========
        // 前序右子树区间：[preorder_left + size_left_subtree + 1, preorder_right]
        // 中序右子树区间：[inorder_root + 1, inorder_right]
        root->right = myBuildTree(preorder, inorder,
                                  preorder_left + size_left_subtree + 1,
                                  preorder_right,
                                  inorder_root + 1,
                                  inorder_right);

        return root;
    }

    TreeNode* buildTree(vector<int>& preorder, vector<int>& inorder) {
        int n = preorder.size();

        // 预处理：构建哈希映射，帮助我们快速定位根节点
        // key: 节点值, value: 该节点在中序遍历中的下标
        for (int i = 0; i < n; ++i) {
            index[inorder[i]] = i;
        }

        // 从整棵树的区间开始递归
        // 前序：[0, n-1]，中序：[0, n-1]
        return myBuildTree(preorder, inorder, 0, n - 1, 0, n - 1);
    }
};
```

---

## 四、参数对照速查表

| 变量名 | 含义 | 图示位置 |
|--------|------|---------|
| `preorder_left` | 当前子树在前序中的左边界 | 前序数组最左边箭头 |
| `preorder_right` | 当前子树在前序中的右边界 | 前序数组最右边箭头 |
| `inorder_left` | 当前子树在中序中的左边界 | 中序数组最左边箭头 |
| `inorder_right` | 当前子树在中序中的右边界 | 中序数组最右边箭头 |
| `preorder_root` | 根节点在前序中的下标 | 前序数组"根"的位置（= `preorder_left`） |
| `inorder_root` | 根节点在中序中的下标 | 中序数组"根"的位置（= `pIndex`） |
| `size_left_subtree` | 左子树的节点个数 | `inorder_root - inorder_left` |

---

## 五、递归过程示例

**输入：**
- `preorder = [3, 9, 20, 15, 7]`
- `inorder  = [9, 3, 15, 20, 7]`

**递归过程（DFS）：**

```
第1层：pre=[0,4], in=[0,4]
        根 = preorder[0] = 3
        在中序中找到 3 在索引 1
        左子树大小 = 1 - 0 = 1
        ├── 左子树: pre=[1,1], in=[0,0]  → 根=9，叶子节点
        └── 右子树: pre=[2,4], in=[2,4]

            第2层(右)：pre=[2,4], in=[2,4]
                      根 = preorder[2] = 20
                      在中序 [15,20,7] 中找到 20 在索引 3（全局）/ 相对索引 1
                      左子树大小 = 3 - 2 = 1
                      ├── 左子树: pre=[3,3], in=[2,2] → 根=15，叶子节点
                      └── 右子树: pre=[4,4], in=[4,4] → 根=7，叶子节点

最终构建的树：

              3
             / \
            9  20
              /  \
             15   7
```

---

## 六、复杂度分析

| 指标 | 复杂度 | 说明 |
|------|--------|------|
| **时间** | O(n) | 每个节点只访问一次；哈希表查找 O(1) |
| **空间** | O(n) | 哈希表 O(n) + 递归栈 O(h)，h 为树的高度 |

> ⚠️ 如果不使用哈希表预处理，每次在中序中线性查找根节点位置，时间复杂度会退化为 **O(n²)**。

---

## 七、易错点 ⚠️

| ❌ 常见错误 | ✅ 正确做法 |
|-----------|-----------|
| 递归终止条件写成 `preorder_left >= preorder_right` | 应该是 `preorder_left > preorder_right`，因为单个节点时 `left == right` 是合法的 |
| 左子树大小算成 `inorder_root` | 应该是 `inorder_root - inorder_left`，即根在中序中的位置减去中序左边界 |
| 右子树前序起点忘记 `+1` | 右子树前序起点是 `preorder_left + size_left_subtree + 1`，要跳过根节点和整个左子树 |
| 哈希表在递归函数里重复构建 | 哈希表只在 `buildTree()` 中**预处理一次**，递归函数只负责查询 |
| 前序和中序的区间边界搞混 | 前序的边界只用于前序数组，中序的边界只用于中序数组，两者通过 `size_left_subtree` 关联 |

---

## 八、记忆口诀 🧠

> **前序取头作根，中序找根分左右。**  
> **左子多长算清楚，前序切分递归走。**  
> **哈希建表查位置，O(1) 查找不犯愁。**

---

## 九、拓展：力扣106（中序 + 后序）

| 题目 | 给定遍历 | 根的位置 | 左子树前序起点 |
|------|---------|---------|--------------|
| **105** | 前序 + 中序 | 前序**第一个** | `preorder_left + 1` |
| **106** | 中序 + 后序 | 后序**最后一个** | `postorder_left` |

两题思路完全一致，区别仅在于**根的位置**和**前序/后序的切分方式**不同。

---

*整理时间：2026-08-03*  
*参考资料：LeetCode 官方题解 + 区间划分图解*
