# LeetCode 19 - 删除链表的倒数第 N 个结点（Remove Nth Node From End of List）

> **标签**：链表、双指针  
> **难度**：中等  
> **核心考点**：快慢指针、虚拟头节点

---

## 一、题目描述

给你一个链表，删除链表的**倒数第 n 个**结点，并且返回链表的头结点。

**示例**：
```
输入: head = [1, 2, 3, 4, 5], n = 2
输出: [1, 2, 3, 5]
解释: 删除倒数第 2 个节点（值为 4 的节点）

输入: head = [1], n = 1
输出: []
解释: 删除唯一的节点，返回空链表

输入: head = [1, 2], n = 1
输出: [1]
解释: 删除倒数第 1 个节点（值为 2 的节点）
```

---

## 二、核心思路：快慢指针

### 2.1 为什么用快慢指针？

要删除倒数第 n 个节点，需要找到**倒数第 n+1 个节点**（即要删节点的前一个），然后修改 `next` 指针。

**关键问题**：链表无法直接知道长度，也无法从后往前遍历。

**快慢指针技巧**：
- `fast` 先走 `n+1` 步
- 然后 `fast` 和 `slow` 同步走，当 `fast` 到末尾时，`slow` 正好在倒数第 `n+1` 个节点

```
链表: 1 → 2 → 3 → 4 → 5, n = 2

步骤1: fast 先走 3 步 (n+1=3)
  fast: dummy → 1 → 2 → 3
  slow: dummy

步骤2: fast 和 slow 同步走
  fast=3, slow=dummy
  fast=4, slow=1
  fast=5, slow=2
  fast=null, slow=3  ← slow 指向要删节点(4)的前一个

步骤3: 删除
  slow->next = slow->next->next  (3 指向 5，跳过 4)

结果: 1 → 2 → 3 → 5
```

### 2.2 为什么 fast 先走 n+1 步？

因为 `slow` 要停在**要删节点的前一个**：
- `fast` 和 `slow` 之间间隔 `n` 个节点
- 当 `fast` 到末尾（`null`）时，`slow` 正好在倒数第 `n+1` 个

---

## 三、代码实现

### 3.1 快慢指针法（推荐）⭐

```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode() : val(0), next(nullptr) {}
 *     ListNode(int x) : val(x), next(nullptr) {}
 *     ListNode(int x, ListNode *next) : val(x), next(next) {}
 * };
 */
class Solution {
public:
    ListNode* removeNthFromEnd(ListNode* head, int n) {
        // 虚拟头节点，统一处理（包括删除头节点的情况）
        ListNode* dummy = new ListNode(0, head);
        ListNode* fast = dummy;
        ListNode* slow = dummy;

        // fast 先走 n+1 步
        for (int i = 0; i <= n; i++) {
            fast = fast->next;
        }

        // fast 和 slow 同步走
        while (fast != nullptr) {
            fast = fast->next;
            slow = slow->next;
        }

        // slow 现在指向要删节点的前一个
        ListNode* toDelete = slow->next;
        slow->next = slow->next->next;

        // 释放内存（LeetCode 不强制要求）
        delete toDelete;

        ListNode* newHead = dummy->next;
        delete dummy;
        return newHead;
    }
};
```

---

## 四、踩坑记录 ⚠️

### ❌ 错误 1：删除头节点时越界

```cpp
// 错误代码（数组法）
int k = N - n + 1;  // 要删节点的位置（从1开始）
for (int i = 0; i < k - 2; i++) {  // 找前一个节点
    cur = cur->next;
}
```

**问题**：当 `k = 1`（删除头节点）时，`k - 2 = -1`，循环不执行，`cur` 还是 `head`。

然后执行：
```cpp
ListNode* temp = cur->next;  // temp = head->next（第二个节点）
cur->next = temp->next;      // 删除的是第二个节点，不是头节点！❌
```

**结果**：该删的没删，不该删的删了。

---

### ❌ 错误 2：删除头节点后返回错误

```cpp
return head;  // 如果 head 被删了，返回的是无效指针！
```

**问题**：头节点被删除后，新的头节点是 `head->next`，但返回的还是原来的 `head`。

**正确做法**：使用虚拟头节点 `dummy`，返回 `dummy->next`，无论删哪个都正确。

---

### ❌ 错误 3：单节点链表崩溃

```cpp
ListNode* temp = cur->next;      // cur->next 可能是 nullptr
cur->next = temp->next;          // temp 是 nullptr，访问 temp->next 崩溃！
```

**问题**：当链表只有一个节点，`n = 1` 时，`cur->next` 是 `nullptr`，`temp` 也是 `nullptr`。

---

### ❌ 错误 4：数组法空间复杂度高

```cpp
vector<ListNode*> v;  // O(n) 额外空间
while (cur != nullptr) {
    v.push_back(cur);  // 存储所有节点
}
```

**问题**：虽然能工作，但空间复杂度 O(n)，不是最优解。

**正确做法**：快慢指针只用两个指针，空间 O(1)。

---

## 五、错误对比总结

```
┌─────────────────────────────────────────────────────────────────────┐
│  ❌ 数组法                              ✅ 快慢指针法                │
│                                                                     │
│  空间: O(n)                              空间: O(1)                 │
│  删除头节点: 特殊判断，容易错              虚拟头节点，统一处理        │
│  单节点: 可能崩溃                         正常处理                   │
│  代码: 复杂                               代码: 简洁                 │
│                                                                     │
│  不推荐 ❌                               推荐 ⭐⭐⭐                  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 六、复杂度分析

| 指标 | 复杂度 | 原因 |
|------|--------|------|
| 时间 | **O(n)** | 遍历链表一次 |
| 空间 | **O(1)** | 只用几个指针 |

---

## 七、易错点 & 注意

| 坑点 | 说明 |
|------|------|
| **虚拟头节点** | 必须加，否则删除头节点需要特殊处理 |
| **fast 先走 n+1 步** | 不是 n 步，要让 slow 停在要删节点的前一个 |
| **循环条件** | `i <= n` 走 n+1 步，不是 `i < n` |
| **返回 dummy->next** | 不是返回 head，head 可能被删 |
| **内存释放** | 删除节点后最好 `delete`，LeetCode 不强制 |

---

## 八、举一反三

| 题号 | 题目 | 关联点 |
|------|------|--------|
| 876 | 链表的中间结点 | 快慢指针，fast 走 2 步 |
| 141 | 环形链表 | 快慢指针判环 |
| 142 | 环形链表 II | 快慢指针找环入口 |
| 160 | 相交链表 | 双指针技巧 |

---

## 九、记忆口诀

> **"删除倒数第 N 个，快慢指针最稳妥；虚拟头节点做锚，fast 先走 N+1 步；同步走到末尾处，slow 正好在前驱"**

---

## 十、面试话术模板

> "这道题我使用快慢指针解决。创建虚拟头节点统一处理，让 fast 先走 n+1 步，然后 fast 和 slow 同步走。当 fast 到达末尾时，slow 正好在要删节点的前一个，修改 next 指针即可。时间 O(n)，空间 O(1)。"

---

*整理日期：2026-07-29*
