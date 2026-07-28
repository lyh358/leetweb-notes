# LeetCode 141 - 环形链表（Linked List Cycle）

> **标签**：链表、双指针、快慢指针  
> **难度**：简单  
> **核心考点**：Floyd 判圈算法（龟兔赛跑）

---

## 一、题目描述

给你一个链表的头节点 `head`，判断链表中是否有环。

如果链表中有某个节点，可以通过连续跟踪 `next` 指针再次到达，则链表中存在环。

**示例**：
```
输入: head = [3, 2, 0, -4], pos = 1
输出: true
解释: 链表中有一个环，其尾部连接到第二个节点（值为 2 的节点）

输入: head = [1, 2], pos = 0
输出: true
解释: 链表中有一个环，其尾部连接到第一个节点（值为 1 的节点）

输入: head = [1], pos = -1
输出: false
解释: 链表中没有环
```

---

## 二、核心思路：快慢指针（Floyd 判圈法）

### 2.1 为什么快慢指针能判环？

想象两个人在环形跑道上跑步：
- **快指针**：每次跑 2 步
- **慢指针**：每次跑 1 步

**如果跑道是环形的**：快指针一定会追上慢指针（套圈相遇）

**如果跑道是直的**：快指针会先跑到终点（遇到 `nullptr`）

```
有环的情况:
3 → 2 → 0 → -4
    ↑_________|

快: 3 → 0 → 2 → -4 → 0 → 2 → ...
慢: 3 → 2 → 0 → -4 → 2 → ...

在某一点，快指针追上慢指针（相遇）

无环的情况:
1 → 2 → 3 → 4 → nullptr

快: 1 → 3 → nullptr
慢: 1 → 2 → 3

快指针先到末尾，说明无环
```

### 2.2 数学证明

假设链表有环，环长度为 `L`：
- 慢指针进入环时，快指针已经在环内某处
- 快指针相对慢指针的速度是 `2 - 1 = 1`（每轮靠近 1 步）
- 环内距离最多 `L-1`，所以最多 `L-1` 轮就会相遇

**结论**：有环必相遇，无环快指针先到末尾。

---

## 三、代码实现

### 3.1 你的代码（快慢指针法）⭐

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
    bool hasCycle(ListNode *head) {
        // 快慢指针法：如果成环，快指针每次走两步，慢指针每次走一步，二者最终肯定相遇
        ListNode* fastnode = head;
        ListNode* slownode = head;

        // 循环跑圈ing：退出条件为快指针碰到末尾的nullptr
        // （用快指针做终止条件是因为它更快，若不成环总会更先遇到nullptr）
        while (fastnode != nullptr) {
            // 快指针走一步
            fastnode = fastnode->next;
            // 若没遇到nullptr再走一步
            if (fastnode != nullptr) {
                fastnode = fastnode->next;
            }
            // 看看是否相遇，相遇则说明有环
            if (fastnode == slownode) {
                return true;
            }
            // 慢指针走一步
            slownode = slownode->next;
        }
        // 循环退出了说明没有环，返回false
        return false;
    }
};
```

### 3.2 更简洁的写法

```cpp
class Solution {
public:
    bool hasCycle(ListNode *head) {
        ListNode* slow = head;
        ListNode* fast = head;

        while (fast != nullptr && fast->next != nullptr) {
            slow = slow->next;          // 慢指针走1步
            fast = fast->next->next;    // 快指针走2步

            if (slow == fast) {         // 相遇，有环
                return true;
            }
        }

        return false;  // 快指针到达末尾，无环
    }
};
```

---

## 四、执行流程可视化

### 有环示例

```
链表: 3 → 2 → 0 → -4
          ↑_________|

初始: slow=3, fast=3

轮次  slow  fast   操作
----  ----  ----   -------------------
 1    2     0      slow+1, fast+2
 2    0     2      slow+1, fast+2 (0→-4→2)
 3    -4    -4     slow+1, fast+2 (2→0→-4)

slow == fast == -4，相遇！返回 true
```

### 无环示例

```
链表: 1 → 2 → 3 → 4 → nullptr

初始: slow=1, fast=1

轮次  slow  fast      操作
----  ----  ----      -------------------
 1    2     3         slow+1, fast+2
 2    3     nullptr   slow+1, fast+2 (3→4→nullptr)

fast == nullptr，循环结束，返回 false
```

---

## 五、复杂度分析

| 指标 | 复杂度 | 原因 |
|------|--------|------|
| 时间 | **O(n)** | 无环时快指针遍历到末尾；有环时最多在环内转几圈就相遇 |
| 空间 | **O(1)** | 只用两个指针 |

---

## 六、易错点 & 注意

| 坑点 | 说明 |
|------|------|
| **循环条件** | 必须判断 `fast != nullptr && fast->next != nullptr`，防止访问空指针 |
| **初始位置** | `slow` 和 `fast` 都从 `head` 开始 |
| **判断相遇时机** | 先移动，再判断相遇（不能先判断再移动，否则初始位置就相等了） |
| **空链表** | `head == nullptr` 时，循环不进入，直接返回 `false`，正确 |
| **单节点无环** | `fast->next == nullptr`，循环不进入，返回 `false`，正确 |

---

## 七、常见错误分析

### ❌ 错误 1：循环条件只判断 fast

```cpp
// 错误！
while (fast != nullptr) {
    fast = fast->next->next;  // 如果 fast->next 是 nullptr，这里越界！
}

// 正确！
while (fast != nullptr && fast->next != nullptr) {
    fast = fast->next->next;
}
```

### ❌ 错误 2：先判断相遇再移动

```cpp
// 错误！初始 slow==fast==head，直接返回 true
while (...) {
    if (slow == fast) return true;  // 一开始就相等了！
    slow = slow->next;
    fast = fast->next->next;
}

// 正确！先移动，再判断
while (...) {
    slow = slow->next;
    fast = fast->next->next;
    if (slow == fast) return true;
}
```

### ❌ 错误 3：快指针走 3 步或更多

```cpp
// 不推荐！虽然也能判环，但可能跳过相遇点
fast = fast->next->next->next;  // 可能直接跳过 slow

// 推荐！走 2 步是最优的，保证一定能相遇
fast = fast->next->next;
```

---

## 八、举一反三

| 题号 | 题目 | 关联点 |
|------|------|--------|
| 142 | 环形链表 II | 找环的入口节点 |
| 287 | 寻找重复数 | 数组视为链表，快慢指针找环 |
| 202 | 快乐数 | 快慢指针判断循环 |
| 876 | 链表的中间结点 | 快慢指针找中点 |
| 19 | 删除链表的倒数第 N 个结点 | 快慢指针找倒数第 k 个 |

### LC 142 找环入口

```cpp
ListNode *detectCycle(ListNode *head) {
    ListNode* slow = head;
    ListNode* fast = head;

    // 第一步：判断是否有环，找到相遇点
    while (fast != nullptr && fast->next != nullptr) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) break;  // 相遇
    }

    if (fast == nullptr || fast->next == nullptr) {
        return nullptr;  // 无环
    }

    // 第二步：找环入口
    // 相遇后，一个指针回到头，两个指针同步走，再次相遇即入口
    ListNode* ptr = head;
    while (ptr != slow) {
        ptr = ptr->next;
        slow = slow->next;
    }

    return ptr;  // 环入口
}
```

**为什么能找入口？**
- 设头到入口距离为 `a`，环长度为 `L`
- 相遇时，慢指针走了 `a + b`，快指针走了 `a + b + nL`
- 快是慢的 2 倍：`2(a+b) = a + b + nL` → `a = nL - b`
- 所以从头和从相遇点同步走，会在入口相遇

---

## 九、记忆口诀

> **"判环用快慢，龟兔来赛跑；有环必相遇，无环先到 nullptr；先走后判断，条件要双保"**

---

## 十、面试话术模板

> "这道题我使用 Floyd 判圈算法（快慢指针）解决。定义两个指针，慢指针每次走一步，快指针每次走两步。如果链表有环，快指针一定会追上慢指针；如果无环，快指针会先到达末尾。时间 O(n)，空间 O(1)。"

---

*整理日期：2026-07-28*
