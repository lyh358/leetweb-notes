# 📘 LeetCode 112 — 路径总和（学习笔记）

---

## 一、题目信息

| 项目 | 内容 |
|------|------|
| 题号 | 112 |
| 难度 | 简单 |
| 标签 | 树、深度优先搜索、二叉树 |
| 核心思路 | DFS 递归 + 减法思想 |

**题目描述**：给定一个二叉树的根节点 `root` 和一个整数 `targetSum`，判断该树中**是否存在**从根节点到叶子节点的路径，这条路径上所有节点值之和等于 `targetSum`。

**关键要求**：
- 路径必须**从根节点开始**
- 路径必须**到叶子节点结束**（左右子节点都为空）
- 只需要判断"是否存在"，不需要找出所有路径

```
输入：root = [5,4,8,11,null,13,4,7,2,null,null,null,1], targetSum = 22

        5
       / \
      4   8
     /   / \
    11  13  4
   / \       \
  7   2       1

路径 5→4→11→2 的和为 22，存在！
输出：true
```

---

## 二、核心原理

### 2.1 减法思想

> **每经过一个节点，targetSum 就减去当前节点的值。**
>
> 当走到**叶子节点**时，如果剩余值等于叶子节点的值，说明找到了一条合法路径。

```
原始 targetSum = 22

走到 5：剩余 22 - 5 = 17
走到 4：剩余 17 - 4 = 13
走到 11：剩余 13 - 11 = 2
走到 2：剩余 2 - 2 = 0 → 叶子节点且剩余为 0 ✅
```

### 2.2 与 LeetCode 437 的区别

| 对比项 | 112（路径总和） | 437（路径总和 III） |
|--------|---------------|-------------------|
| **起点** | 必须从根开始 | 任意节点都可以 |
| **终点** | 必须是叶子节点 | 任意节点都可以 |
| **返回值** | bool（是否存在） | int（路径数量） |
| **递归层数** | 单层 DFS | 双重 DFS |
| **难度** | 简单 ⭐ | 中等 ⭐⭐⭐ |

> 112 是 437 的"简化版"：起点固定 + 终点固定 + 只需判断存在性。

---

## 三、完整代码（含详细注释）

```cpp
class Solution {
public:
    /**
     * 判断是否存在从 root 出发、和为 targetSum 的根到叶子路径
     * @param root 当前子树的根节点
     * @param targetSum 从当前节点到叶子节点还需要凑够的值
     * @return true/false 是否存在这样的路径
     */
    bool hasPathSum(TreeNode* root, int targetSum) {
        // ===== 递归出口 1：空节点 =====
        // 空树不可能有路径，返回 false
        if (!root) return false;

        // ===== 递归出口 2：叶子节点 =====
        // 左右子节点都为空，说明是叶子节点
        // 此时判断：剩余目标值是否等于当前节点的值
        // 如果相等，说明从根到该叶子的路径和正好等于 targetSum
        if (root->left == nullptr && root->right == nullptr) {
            return targetSum == root->val;
        }

        // ===== 递归分解 =====
        // 当前节点不是叶子，继续向左右子树搜索
        // 每递归一层，targetSum 减去当前节点的值
        // 只要左子树或右子树中任意一条路径满足条件即可（|| 短路求值）
        return hasPathSum(root->left, targetSum - root->val) 
            || hasPathSum(root->right, targetSum - root->val);
    }
};
```

---

## 四、逐行代码解析

### 4.1 空节点判断

```cpp
if (!root) return false;
```

- 如果当前节点为空，说明这条路走不通（走到了空子树）
- 空节点不是叶子节点，不能作为路径终点
- **注意**：这个判断必须放在最前面，否则后面访问 `root->left` 会空指针

### 4.2 叶子节点判断

```cpp
if (root->left == nullptr && root->right == nullptr) {
    return targetSum == root->val;
}
```

- **叶子节点定义**：左右子节点都为空的节点
- **判断逻辑**：走到叶子时，剩余目标值 `targetSum` 应该正好等于当前节点的值
  - 如果相等：`true`（找到一条合法路径）
  - 如果不相等：`false`（这条路径的和不对）

### 4.3 递归搜索

```cpp
return hasPathSum(root->left, targetSum - root->val) 
    || hasPathSum(root->right, targetSum - root->val);
```

- **`targetSum - root->val`**：走到子节点时，剩余目标值减少了当前节点的值
- **`||`**：只要左子树或右子树中存在合法路径，就返回 `true`（短路求值）

---

## 五、执行流程图解

以示例树为例，`targetSum = 22`：

```
        5
       / \
      4   8
     /   / \
    11  13  4
   / \       \
  7   2       1
```

### 递归调用栈展开

```
hasPathSum(5, 22)
  ├── 5 不是叶子
  ├── hasPathSum(4, 22-5=17)
  │     ├── 4 不是叶子（左子 11，右子 null）
  │     ├── hasPathSum(11, 17-4=13)
  │     │     ├── 11 不是叶子
  │     │     ├── hasPathSum(7, 13-11=2)
  │     │     │     ├── 7 是叶子！
  │     │     │     ├── 2 == 7? ❌ false
  │     │     │     └── 返回 false
  │     │     ├── hasPathSum(null, ...) → false（|| 短路，但左边已 false，继续右边）
  │     │     ├── 7 的右子树是 null → false
  │     │     └── 返回 false || false = false
  │     │
  │     ├── hasPathSum(null, ...) → false
  │     └── 返回 false || false = false
  │
  ├── hasPathSum(8, 22-5=17)
  │     ├── 8 不是叶子
  │     ├── hasPathSum(13, 17-8=9)
  │     │     ├── 13 是叶子！
  │     │     ├── 9 == 13? ❌ false
  │     │     └── 返回 false
  │     ├── hasPathSum(4, 17-8=9)
  │     │     ├── 4 不是叶子
  │     │     ├── hasPathSum(null, ...) → false
  │     │     ├── hasPathSum(1, 9-4=5)
  │     │     │     ├── 1 是叶子！
  │     │     │     ├── 5 == 1? ❌ false
  │     │     │     └── 返回 false
  │     │     └── 返回 false || false = false
  │     └── 返回 false || false = false
  │
  └── 返回 false || false = false ❌

等等，结果应该是 true！让我重新检查...

啊！我漏了一条路径：

hasPathSum(5, 22)
  ├── hasPathSum(4, 17)
  │     ├── hasPathSum(11, 13)
  │     │     ├── hasPathSum(7, 2) → 7 是叶子，2==7? ❌ false
  │     │     ├── hasPathSum(2, 2) → 2 是叶子！2==2? ✅ true！
  │     │     └── 返回 true！
  │     └── 返回 true（|| 短路，右边不执行）
  └── 返回 true ✅
```

### 找到的路径

```
5 → 4 → 11 → 2

验证：5 + 4 + 11 + 2 = 22 ✅
```

### 递归树示意图

```
                    hasPathSum(5, 22)
                   /                  \
                  /                    \
        hasPathSum(4, 17)        hasPathSum(8, 17)
              /      \                /        \
             /        \              /          \
    hasPathSum(11,13)  false   hasPathSum(13,9)  hasPathSum(4,9)
       /      \                    =false          /      \
      /        \                                /        \
  hasPathSum  hasPathSum                    false    hasPathSum(1,5)
  (7,2)      (2,2)                                    =false
  =false      =true

              ↑
        2 是叶子，2==2 → true
```

---

## 六、递归逻辑一句话总结

> **走到叶子节点时检查：剩余目标值是否等于当前节点值。不是叶子就继续走，每走一步 targetSum 减去当前值。只要左右子树中有一条路通，就返回 true。**

---

## 七、易错点总结

| 坑点 | 说明 |
|------|------|
| ❌ 叶子节点判断条件写错 | 必须是 `left == nullptr && right == nullptr`，不能只判断 `!left` |
| ❌ 空节点返回 true | 空节点不是叶子，应该返回 `false`。如果返回 `true`，会把空子树误当成合法路径 |
| ❌ 减法顺序搞反 | 是 `targetSum - root->val`，不是 `root->val - targetSum` |
| ❌ 忘记叶子节点判断 | 如果不判断叶子节点，走到任意节点都可能返回 true，不要求到叶子 |
| ❌ 用 `&&` 代替 `||` | 必须左右子树**任意一个**满足即可，用 `||` 不是 `&&` |
| ❌ 与 437 搞混 | 112 要求**根到叶子**，437 要求**任意起点任意终点**，不要混用 |

---

## 八、相关题目

| 题号 | 题目 | 关联点 |
|------|------|--------|
| 113 | 路径总和 II | 112 的升级版，要求返回所有路径（需要回溯） |
| 437 | 路径总和 III | 112 的升级版，起点和终点都不固定（双重 DFS） |
| 124 | 二叉树中的最大路径和 | 路径可以不走到底，需要后序遍历 |
| 257 | 二叉树的所有路径 | 返回所有根到叶子的路径字符串 |

---

## 九、一句话总结

> **112 = 根到叶子的减法 DFS。每走一步 targetSum 减当前值，走到叶子时看剩余值是否等于叶子值。`||` 短路求值让代码简洁优雅。这是理解 113 和 437 的基础！**
