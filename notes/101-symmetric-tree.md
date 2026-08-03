# 力扣 101：对称二叉树学习笔记

## 一、 题目核心解析

**题目描述**：给你一个二叉树的根节点 `root`，检查它是否轴对称。
**核心定义**：判断一棵树是否轴对称，等价于判断这棵树的**左子树**和**右子树**是否互为镜像。

**关键示例**：
```text
      1
     / \
    2   2
   / \ / \
  3  4 4  3
```
左子树和右子树呈完美的镜像关系，因此返回 `true`。

---

## 二、 核心思想：如何透彻理解递归？

对称二叉树的递归比前几道题稍微复杂一点，因为它不再是“自己和自己比”，而是**“两个节点同时向下走”**。

**思维转换**：
不要只盯着一个节点看，你要**同时拿出两个节点**（比如左子树的节点 A 和右子树的节点 B），问自己：**“这两个节点互为镜像，需要满足什么条件？”**

答案有两个：
1. **值相等**：A 的值必须等于 B 的值。
2. **子树互为镜像**：A 的左孩子必须和 B 的右孩子互为镜像；A 的右孩子必须和 B 的左孩子互为镜像。

**递归三步法**：
1. **确定递归出口（边界条件）**：
   - 两个节点都为空：说明对称位置都走到底了，返回 `true`。
   - 一个为空，一个不为空：说明结构不对称，返回 `false`。
   - 两个都不为空，但值不相等：说明数值不对称，返回 `false`。
2. **单层递归逻辑**：
   - 比较**外侧**：左节点的左孩子 vs 右节点的右孩子。
   - 比较**内侧**：左节点的右孩子 vs 右节点的左孩子。
3. **结果合并**：
   - 只有当“外侧对称” **且** “内侧对称”同时成立时，这两个节点才是真正对称的。

> **顿悟时刻**：把整棵树劈成两半，左半边向右下走，右半边向左下走。只要它们一路上遇到的节点值都相等，且最终同时走到空节点，这棵树就是对称的。

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
    bool isSymmetric(TreeNode* root) {
        if (root == nullptr) return true;
        // 从根节点的左右孩子开始比较
        return compare(root->left, root->right);
    }

private:
    bool compare(TreeNode* left, TreeNode* right) {
        // 1. 递归出口：处理空节点和值不相等的情况
        if (left == nullptr && right == nullptr) return true;
        if (left == nullptr || right == nullptr) return false;
        if (left->val != right->val) return false;
        
        // 2. 单层递归逻辑：比较外侧和内侧
        bool outside = compare(left->left, right->right);   // 外侧：左左 vs 右右
        bool inside = compare(left->right, right->left);    // 内侧：左右 vs 右左
        
        // 3. 结果合并：内外侧都必须对称
        return outside && inside;
    }
};
```

---

## 四、 复杂度分析

- **时间复杂度：$O(n)$**
  其中 $n$ 为二叉树的节点总数。每个节点在递归中最多被访问一次。
- **空间复杂度：$O(h)$**
  其中 $h$ 为二叉树的高度。空间消耗主要来自于递归调用栈的深度。最坏情况下（树退化为链表），空间复杂度为 $O(n)$。

---

## 五、 拓展思路：迭代法（队列模拟）

如果你觉得“同时比较两个节点”的递归有些抽象，可以用**队列（BFS）**来模拟这个过程，逻辑非常直观。

**核心思路**：
每次从队列中**连续弹出两个节点**进行比较。如果它们相等，就把它们的子节点按“镜像顺序”（左左、右右、左右、右左）重新压入队列，等待下一轮比较。

**参考代码**：
```cpp
class Solution {
public:
    bool isSymmetric(TreeNode* root) {
        if (root == nullptr) return true;
        
        queue<TreeNode*> q;
        q.push(root->left);
        q.push(root->right);
        
        while (!q.empty()) {
            TreeNode* leftNode = q.front(); q.pop();
            TreeNode* rightNode = q.front(); q.pop();
            
            // 两个都为空，继续比较下一对
            if (!leftNode && !rightNode) continue;
            // 一个空一个不空，或者值不相等，直接返回 false
            if (!leftNode || !rightNode || leftNode->val != rightNode->val) {
                return false;
            }
            
            // 按镜像顺序入队
            q.push(leftNode->left);   // 左左
            q.push(rightNode->right); // 右右
            q.push(leftNode->right);  // 左右
            q.push(rightNode->left);  // 右左
        }
        return true;
    }
};
```

---

## 六、 学习总结

1. **递归视角的转换**：从“单节点遍历”升级为“双节点同步比较”。这是解决对称、相同树等问题的核心思维。
2. **边界条件的严谨性**：在双节点比较中，`null` 的处理是重中之重。记住口诀：**双空为真，一空一非空为假，非空比数值**。
3. **DFS vs BFS 对比**：
   - **DFS（递归）**：代码优雅，逻辑自洽，完美契合“外侧+内侧”的对称定义。
   - **BFS（队列）**：将抽象的对称具象化为“成对入队、成对出队”，适合在脑海中建立物理模型。

---

**下一步建议**：对称二叉树（101）和翻转二叉树（226）是二叉树递归的“双子星”。建议你闭上眼睛，在脑海中分别画出这两道题的递归展开图，你会发现它们的底层逻辑是完全相通的！

---

# 1. 定义“黑盒”：我到底写了一个什么函数？

这个思路被称为“黑盒思维”（Black Box Thinking），正是掌握递归最核心的心法。在算法界，这被称为**“递归信任假设”**。

用这个思路来重新理解力扣 101（对称二叉树），原本复杂的嵌套逻辑就会变得极其清晰。

### 1. 定义“黑盒”：我到底写了一个什么函数？

首先，我们要明确这个递归函数的契约（Contract）：
* **函数名**：`compare`
* **输入（Input）**：两个节点 `left` 和 `right`。
* **输出（Output）**：一个布尔值 `bool`。
* **功能（Function）**：判断以 `left` 为根的树，和以 `right` 为根的树，是否互为镜像。

### 2. 核心心法：无条件相信这个“黑盒”

按照这个思路，我们在写代码时，**绝对不要**去想象 `compare` 函数内部是怎么一层层往下递归的。我们只需要站在当前这一层，像调用一个已经写好的完美工具一样去调用它：

* 当我写下 `compare(left->left, right->right)` 时，我**完全相信**它能帮我判断出“左树的左子树”和“右树的右子树”是不是镜像，并且它会乖乖地返回一个 `true` 或 `false` 给我。
* 同理，当我写下 `compare(left->right, right->left)` 时，我也**完全相信**它能判断出“左树的右子树”和“右树的左子树”是不是镜像，并返回结果。

既然这两个工具都能完美工作，那么我只需要把它们的返回值用 `&&`（逻辑与）连起来，就得到了当前这一层的结果。

### 3. 当前层需要干什么？（处理边界）

既然子问题（黑盒）已经完美解决了，那么作为当前层的代码，我只需要处理**“黑盒”无法处理的特殊情况**（即递归出口）：

* **情况 A**：如果 `left` 和 `right` 都是空节点（`nullptr`），说明这两个位置对称地走到了尽头，直接返回 `true`。
* **情况 B**：如果 `left` 和 `right` 只有一个为空，说明一边有节点一边没节点，结构不对称，直接返回 `false`。
* **情况 C**：如果都不为空，但 `left->val != right->val`，说明当前节点的值就不一样，直接返回 `false`。

只有当以上三种“必死”的情况都排除了，我才会放心地把剩下的深层比较工作，丢给那个我“完全相信”的黑盒函数去处理。

### 4. 代码实现（贯彻“黑盒”思维）

```cpp
class Solution {
public:
    bool isSymmetric(TreeNode* root) {
        if (!root) return true;
        return compare(root->left, root->right);
    }

private:
    bool compare(TreeNode* left, TreeNode* right) {
        // 【当前层职责】：处理黑盒无法处理的边界情况
        if (!left && !right) return true;       // 双空：对称
        if (!left || !right) return false;      // 一空一非空：不对称
        if (left->val != right->val) return false; // 值不等：不对称
        
        // 【黑盒调用】：我完全相信这两个调用能返回正确的子树比较结果
        // 并且我只需要把它们的结果合并
        return compare(left->left, right->right) && compare(left->right, right->left);
    }
};
```

### 5. 总结

这个理解方式，完美避开了新手最容易犯的“在脑子里模拟递归栈”的错误。写递归就像**套娃**：你不需要知道最里面的小套娃是怎么制造的，你只需要知道，只要你给了它正确的输入，它就一定会给你一个符合预期的输出。掌握了这个思维，以后所有的树形递归题（比如路径总和、二叉树序列化）对你来说都只是修改一下输入输出和边界条件而已！