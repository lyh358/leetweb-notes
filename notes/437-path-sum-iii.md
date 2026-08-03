## 二、核心原理

### 2.1 问题拆解

题目说"路径不需要从根开始"，这意味着：**每个节点都可能是某条路径的起点**。

所以问题可以拆成两步：
1. **枚举起点**：遍历每个节点，把它当作路径的起点
2. **向下搜索**：从该起点出发，向下走（只能走子节点），统计路径和等于 target 的数量

### 2.2 为什么是"双重 DFS"？

| 第一层 DFS | 第二层 DFS |
|-----------|-----------|
| 遍历每个节点 | 从当前节点出发，向下搜索 |
| 决定"从哪开始" | 决定"怎么走" |
| `pathSum(root)` | `countFrom(root, target)` |

---

## 三、解法一：优雅的双重递归

### 3.1 完整代码（含详细注释）

```cpp
class Solution {
public:
    /**
     * 第一层 DFS：枚举每个节点作为起点
     * 返回整棵树中，和为 targetSum 的路径总数
     */
    int pathSum(TreeNode* root, int targetSum) {
        // 空树没有路径
        if (!root) return 0;

        // 三部分之和：
        // 1. 以当前节点为起点的路径数
        // 2. 以左子树中某节点为起点的路径数
        // 3. 以右子树中某节点为起点的路径数
        return countFrom(root, targetSum) 
             + pathSum(root->left, targetSum) 
             + pathSum(root->right, targetSum);
    }

    /**
     * 第二层 DFS：以 root 为起点，向下搜索
     * 返回以 root 为起点、和为 targetSum 的路径数量
     * 
     * 核心思想：每走一步，targetSum 就减去当前节点的值
     * 当 targetSum == 当前节点值时，说明找到了一条路径
     */
    int countFrom(TreeNode* root, long long targetSum) {
        // 空节点，没有路径
        if (!root) return 0;

        int count = 0;
        // 当前节点的值正好等于剩余的 targetSum，找到一条路径
        if (root->val == targetSum) count++;

        // 继续向左右子树搜索，targetSum 减去当前节点的值
        count += countFrom(root->left, targetSum - root->val);
        count += countFrom(root->right, targetSum - root->val);

        return count;
    }
};
```

### 3.2 逐行解析

#### `pathSum` 函数（第一层 DFS）

```cpp
return countFrom(root, targetSum)           // ① 以当前节点为起点
     + pathSum(root->left, targetSum)      // ② 以左子树某节点为起点
     + pathSum(root->right, targetSum);     // ③ 以右子树某节点为起点
```

**为什么要拆成三部分？**

因为路径的起点可以是：
- 当前节点 `root`
- 左子树中的某个节点
- 右子树中的某个节点

这三部分互不重叠，加起来就是全部路径。

#### `countFrom` 函数（第二层 DFS）

```cpp
if (root->val == targetSum) count++;
```

**为什么只判断 `==`？**

因为我们每递归一层，就把 `targetSum` 减去了当前节点的值：

```
原始 targetSum = 8

走到节点 5：targetSum 还是 8，5 != 8，继续
  走到节点 3：targetSum 变成 8-5=3，3 == 3 ✅ 找到一条路径！

走到节点 5：targetSum 还是 8，5 != 8，继续
  走到节点 2：targetSum 变成 8-5=3，2 != 3，继续
    走到节点 1：targetSum 变成 3-2=1，1 == 1 ✅ 找到一条路径！
```

**减法思想**：每经过一个节点，剩余的目标和就减少，当剩余值等于当前节点值时，说明从起点到当前节点的路径和正好等于 target。

### 3.3 执行流程图解

以示例树为例，`targetSum = 8`：

```
        10
       /  \
      5   -3
     / \    \
    3   2   11
   / \   \
  3  -2   1
```

#### 第一层：pathSum(10)

```
pathSum(10)
  ├── countFrom(10, 8)     → 10 != 8，左右递归...
  │     ├── countFrom(5, 8-10=-2)  → 5 != -2
  │     └── countFrom(-3, 8-10=-2) → -3 != -2
  │     └── 返回 0
  │
  ├── pathSum(5)           → 递归处理左子树
  │     ├── countFrom(5, 8)
  │     │     ├── 5 == 8? 否
  │     │     ├── countFrom(3, 8-5=3)
  │     │     │     ├── 3 == 3? ✅ count=1
  │     │     │     ├── countFrom(3, 3-3=0) → 3==0? 否
  │     │     │     │     ├── countFrom(null, ...) → 0
  │     │     │     │     └── countFrom(-2, ...) → -2==0? 否 → 0
  │     │     │     │     └── 返回 0
  │     │     │     └── countFrom(-2, 3-3=0) → -2==0? 否 → 0
  │     │     │     └── 返回 1
  │     │     ├── countFrom(2, 8-5=3)
  │     │     │     ├── 2 == 3? 否
  │     │     │     ├── countFrom(null, ...) → 0
  │     │     │     └── countFrom(1, 3-2=1)
  │     │     │           ├── 1 == 1? ✅ count=1
  │     │     │           └── 返回 1
  │     │     │     └── 返回 1
  │     │     └── 返回 1+1 = 2
  │     │
  │     ├── pathSum(3) → countFrom(3,8)=0 + pathSum(3的左)=0 + pathSum(3的右)=0 → 0
  │     └── pathSum(2) → countFrom(2,8)=0 + pathSum(2的左)=0 + pathSum(2的右)=0 → 0
  │     └── 返回 2 + 0 + 0 = 2
  │
  └── pathSum(-3)          → 递归处理右子树
        ├── countFrom(-3, 8)
        │     ├── -3 == 8? 否
        │     ├── countFrom(null, ...) → 0
        │     └── countFrom(11, 8-(-3)=11)
        │           ├── 11 == 11? ✅ count=1
        │           └── 返回 1
        │     └── 返回 1
        ├── pathSum(null) → 0
        └── pathSum(null) → 0
        └── 返回 1 + 0 + 0 = 1

最终结果：0 + 2 + 1 = 3 ✅
```

#### 找到的三条路径

| 路径 | 节点序列 | 和 | 在哪被发现 |
|------|---------|-----|-----------|
| ① | 5 → 3 | 5+3=8 | countFrom(5,8) → countFrom(3,3) |
| ② | 5 → 2 → 1 | 5+2+1=8 | countFrom(5,8) → countFrom(2,3) → countFrom(1,1) |
| ③ | -3 → 11 | -3+11=8 | countFrom(-3,8) → countFrom(11,11) |

---

## 四、解法二：另一种写法（累加和）

### 4.1 代码

```cpp
class Solution {
public:
    int ans = 0;
    int target;

    /**
     * 第二层 DFS：从 root 出发，累加路径和
     * @param sum 从起点到当前父节点的路径和
     */
    void dfs(TreeNode* root, long long sum) {
        if (!root) return;

        sum += root->val;           // 累加当前节点的值
        if (sum == target) ans++;  // 路径和等于 target，找到一条路径

        dfs(root->left, sum);      // 继续向左走
        dfs(root->right, sum);     // 继续向右走
    }

    /**
     * 第一层 DFS：枚举起点
     */
    int pathSum(TreeNode* root, int targetSum) {
        target = targetSum;
        if (!root) return 0;

        dfs(root, 0);                    // 以当前节点为起点
        pathSum(root->left, targetSum);   // 以左子树节点为起点
        pathSum(root->right, targetSum);  // 以右子树节点为起点

        return ans;
    }
};
```

### 4.2 两种写法的对比

| 对比项 | 解法一（减法） | 解法二（累加） |
|--------|-------------|--------------|
| **核心思想** | 每走一步，target 减去当前值 | 每走一步，sum 加上当前值 |
| **判断条件** | `root->val == targetSum` | `sum == target` |
| **递归参数** | `targetSum - root->val` | `sum + root->val` |
| **返回值** | `countFrom` 返回路径数 | `dfs` 用成员变量 `ans` 计数 |
| **代码优雅度** | ⭐⭐⭐⭐⭐ 更优雅，函数有返回值 | ⭐⭐⭐ 需要成员变量 |
| **推荐度** | **推荐** | 理解即可 |

**两种写法本质完全相同**，只是"正向累加"和"反向减法"的区别。

---

## 五、复杂度分析

| 指标 | 值 | 说明 |
|------|------|------|
| **时间** | $O(n^2)$ 最坏 | 对每个节点（n 个）做一次 DFS，每次 DFS 最坏遍历整棵树 |
| **空间** | $O(h)$ | 递归栈深度，h 为树高。最坏退化为链表时 $O(n)$ |

> 最坏情况：所有节点值都是 0，targetSum = 0，此时每个起点都要遍历到底，时间 $O(n^2)$。

---

## 六、前缀和优化（进阶）

### 6.1 核心思想

> **路径和可以转化为前缀和的差。**
>
> 如果 `prefix[j] - prefix[i] = target`，说明从节点 i+1 到节点 j 的路径和等于 target。
>
> 用哈希表记录前缀和出现的次数，可以把时间降到 $O(n)$。

### 6.2 代码

```cpp
class Solution {
public:
    unordered_map<long long, int> prefix;  // 前缀和 -> 出现次数
    int target;
    int ans = 0;

    void dfs(TreeNode* root, long long sum) {
        if (!root) return;

        sum += root->val;

        // 如果存在前缀和等于 sum - target，说明中间有一段路径和为 target
        ans += prefix[sum - target];

        // 当前前缀和入表
        prefix[sum]++;

        dfs(root->left, sum);
        dfs(root->right, sum);

        // 回溯：离开当前节点时，前缀和出表
        prefix[sum]--;
    }

    int pathSum(TreeNode* root, int targetSum) {
        target = targetSum;
        prefix[0] = 1;  // 前缀和为0出现1次（空路径）
        dfs(root, 0);
        return ans;
    }
};
```

### 6.3 复杂度对比

| 方法 | 时间 | 空间 |
|------|------|------|
| 双重 DFS | $O(n^2)$ | $O(h)$ |
| 前缀和优化 | **$O(n)$** | $O(n)$ |

---

## 七、易错点总结

| 坑点 | 说明 |
|------|------|
| ❌ 只用一层 DFS | 只从根节点出发搜索，会漏掉不以根为起点的路径 |
| ❌ 路径方向搞错 | 必须是**向下**的（父→子），不能回头走 |
| ❌ 用 `int` 存 sum | 节点值和 target 可能很大，累加会溢出，用 `long long` |
| ❌ 空节点没处理 | `if (!root) return 0;` 必须写 |
| ❌ 解法二中 `ans` 没重置 | 如果多次调用 pathSum，成员变量 `ans` 要清零 |
| ❌ 前缀和优化忘记回溯 | `prefix[sum]--` 必须在递归返回后执行，否则影响其他分支 |
| ❌ 前缀和优化忘记 `prefix[0]=1` | 空路径的前缀和为 0，必须初始化为 1 |

---

## 八、一句话总结

> **路径总和 III = 双重 DFS：第一层枚举每个节点作为起点，第二层从该起点向下搜索。减法思想是"每走一步 target 减当前值"，累加思想是"每走一步 sum 加当前值"。双重 DFS 时间 $O(n^2)$，前缀和优化可降到 $O(n)$。**
