# LeetCode 160 - 相交链表（Intersection of Two Linked Lists）

> **标签**：链表、双指针、哈希表  
> **难度**：简单  
> **核心考点**：双指针找交点、链表长度差

---

## 一、题目描述

给你两个单链表的头节点 `headA` 和 `headB`，请你找出并返回两个单链表相交的起始节点。如果两个链表不存在相交节点，返回 `null`。

**注意**：
- 函数返回结果后，链表必须**保持其原始结构**
- 可以假定整个链表结构中没有循环

**示例**：
```
输入: intersectVal = 8, listA = [4,1,8,4,5], listB = [5,6,1,8,4,5], skipA = 2, skipB = 3
输出: 相交节点值为 8 的节点
解释: 链表 A 的前两个节点是 4→1，链表 B 的前三个节点是 5→6→1，
     从 8 开始两个链表共享后面的 8→4→5
```

---

## 二、核心思路

### 2.1 相交链表的性质

两个链表相交后，**从相交点开始后面的节点完全相同**（因为链表节点只有一个 `next` 指针）。

```
A:    4 → 1
           ↘
            8 → 4 → 5 → null
           ↗
B:    5 → 6 → 1

相交节点是值为 8 的节点
```

### 2.2 方法一：哈希表法（你的代码）

遍历链表 A，把所有节点存入哈希表；再遍历链表 B，第一个在哈希表中的节点就是交点。

### 2.3 方法二：双指针法 ⭐ 最优

让两个指针分别从两个链表头出发，走到末尾后切换到另一个链表头，最终会在交点相遇。

**为什么能相遇？**

设链表 A 长度为 `a + c`，链表 B 长度为 `b + c`，其中 `c` 是公共部分。

- 指针 A 走：`a + c + b`（A 链表 + 公共部分 + B 链表非公共部分）
- 指针 B 走：`b + c + a`（B 链表 + 公共部分 + A 链表非公共部分）

两者走的总长度相同，都是 `a + b + c`，所以一定同时到达交点！

```
A: 4 → 1 → 8 → 4 → 5 → null → 5 → 6 → 1 → 8 → 4 → 5 → null
   ↑_________________________↑
   指针 A 走到 null 后切换到 headB

B: 5 → 6 → 1 → 8 → 4 → 5 → null → 4 → 1 → 8 → 4 → 5 → null
   ↑_________________________↑
   指针 B 走到 null 后切换到 headA

两者在第一个 8 处相遇！
```

---

## 三、代码实现

### 3.1 哈希表法（你的代码）

```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode(int x) : val(x), next(NULL) {}
 * };
 */
class Solution {
public:
    ListNode *getIntersectionNode(ListNode *headA, ListNode *headB) {
        unordered_set<ListNode*> uset;
        ListNode *cur = headA;

        // 遍历链表 A，把所有节点存入哈希表
        while (cur != nullptr) {
            uset.insert(cur);
            cur = cur->next;
        }

        // 遍历链表 B，找第一个在哈希表中的节点
        cur = headB;
        while (cur != nullptr) {
            if (uset.count(cur)) {   // 为什么用 count 不用 find？
                return cur;
            }
            cur = cur->next;
        }

        return nullptr;
    }
};
```

### 3.2 双指针法 ⭐ 最优

```cpp
class Solution {
public:
    ListNode *getIntersectionNode(ListNode *headA, ListNode *headB) {
        ListNode *pA = headA;
        ListNode *pB = headB;

        while (pA != pB) {
            // A 走到末尾后切换到 B 的头
            pA = (pA == nullptr) ? headB : pA->next;
            // B 走到末尾后切换到 A 的头
            pB = (pB == nullptr) ? headA : pB->next;
        }

        return pA;  // 相交返回交点，不相交返回 null（同时到达 null）
    }
};
```

---

## 四、为什么用 `uset.count()` 而不是 `uset.find()`？

### 4.1 两者功能对比

| 方法 | 返回值 | 用法 |
|------|--------|------|
| `uset.find(key)` | 返回**迭代器**（找到）或 `uset.end()`（未找到） | 需要迭代器时用 |
| `uset.count(key)` | 返回**整数**（0 或 1） | 只需要判断是否存在时用 |

### 4.2 你的场景

你只需要判断 "节点是否在集合中"，不需要知道具体位置：

```cpp
// 用 count：简洁，直接返回 bool 语义
if (uset.count(cur)) {
    return cur;
}

// 用 find：需要比较迭代器
if (uset.find(cur) != uset.end()) {
    return cur;
}
```

### 4.3 为什么推荐用 count？

1. **代码更简洁**：不需要写 `!= uset.end()`
2. **语义清晰**：`count` 表示"计数"，对于 `unordered_set` 只有 0 或 1，直接表示"存在/不存在"
3. **对于 set 没有性能差异**：`count` 和 `find` 底层都是哈希查找，时间都是 O(1)

### 4.4 什么时候必须用 find？

当你需要获取元素的**迭代器**（比如要删除、修改）时：

```cpp
auto it = uset.find(cur);
if (it != uset.end()) {
    uset.erase(it);  // 需要迭代器才能删除
}
```

> 对于本题，只需要判断存在性，`count` 更简洁；`find` 也能用，只是代码稍长。

---

## 五、复杂度分析

| 方法 | 时间复杂度 | 空间复杂度 | 说明 |
|------|-----------|-----------|------|
| 哈希表法 | **O(m + n)** | **O(m)** | m 和 n 是两个链表长度，需要存一个链表 |
| 双指针法 | **O(m + n)** | **O(1)** | 最优，不需要额外空间 |

---

## 六、执行流程可视化

### 哈希表法

```
A: 4 → 1 → 8 → 4 → 5 → null
B: 5 → 6 → 1 → 8 → 4 → 5 → null

步骤1: 遍历 A，存入哈希表
  uset = {4, 1, 8, 4, 5}（节点地址）

步骤2: 遍历 B
  5: 不在 uset
  6: 不在 uset
  1: 不在 uset（注意：这个 1 和 A 中的 1 是不同节点！）
  8: 在 uset！返回 8
```

### 双指针法

```
A: 4 → 1 → 8 → 4 → 5 → null
B: 5 → 6 → 1 → 8 → 4 → 5 → null

pA: 4 → 1 → 8 → 4 → 5 → null → 5 → 6 → 1 → 8 ✓
pB: 5 → 6 → 1 → 8 → 4 → 5 → null → 4 → 1 → 8 ✓

两者在 8 相遇！
```

---

## 七、易错点 & 注意

| 坑点 | 说明 |
|------|------|
| **比较的是节点地址** | 不是节点值！值相同但节点不同不算相交 |
| **双指针切换条件** | `pA == nullptr` 时切换到 `headB`，不是 `pA->next == nullptr` |
| **不相交的情况** | 双指针最后会同时到达 `null`，返回 `null` |
| **count vs find** | 只需要判断存在性时用 `count` 更简洁 |

---

## 八、举一反三

| 题号 | 题目 | 关联点 |
|------|------|--------|
| 141 | 环形链表 | 快慢指针 |
| 142 | 环形链表 II | 快慢指针找环入口 |
| 21 | 合并两个有序链表 | 双指针遍历两个链表 |

---

## 九、记忆口诀

> **"相交链表找交点，哈希双针都能行；哈希存一判另一，双针走完全程遇；count 简洁 find 取迭代，存在判断 count 赢"**

---

## 十、面试话术模板

> "这道题我提供两种解法：哈希表法和双指针法。哈希表法遍历一个链表存入集合，再遍历另一个找第一个存在的节点，时间 O(m+n)，空间 O(m)。双指针法让两个指针分别遍历两个链表，走到末尾后切换到另一个链表头，最终会在交点相遇，时间 O(m+n)，空间 O(1)，是更优解。"

---

*整理日期：2026-07-28*
