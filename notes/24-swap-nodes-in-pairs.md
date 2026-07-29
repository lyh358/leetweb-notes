# LeetCode 24 - 两两交换链表中的节点（Swap Nodes in Pairs）

> **标签**：链表、递归、迭代  
> **难度**：中等  
> **核心考点**：链表指针操作、递归思想

---

## 一、题目描述

给你一个链表，两两交换其中相邻的节点，并返回交换后链表的头节点。

**你必须在不修改节点内部的值的情况下完成本题**（即，只能进行节点交换）。

**示例**：
```
输入: head = [1, 2, 3, 4]
输出: [2, 1, 4, 3]

输入: head = []
输出: []

输入: head = [1]
输出: [1]
```

---

## 二、核心思路

### 2.1 问题分析

两两交换相邻节点，即第1个和第2个交换，第3个和第4个交换，以此类推。

```
原链表: 1 → 2 → 3 → 4 → 5 → 6
         ↓交换   ↓交换   ↓交换
结果:   2 → 1 → 4 → 3 → 6 → 5
```

**关键**：交换两个节点需要修改3个指针关系，容易搞混，建议画图。

---

## 三、方法一：迭代法（哑节点）⭐ 推荐

### 3.1 思路

使用**哑节点**统一处理，用 `prev` 指针指向待交换节点对的前一个节点。

```
初始: dummy → 1 → 2 → 3 → 4
      ↑
     prev

交换 1 和 2:
  1. prev->next = 2        (dummy → 2)
  2. 1->next = 2->next     (1 → 3)
  3. 2->next = 1           (2 → 1)

  结果: dummy → 2 → 1 → 3 → 4
              ↑
             prev 移动到 1

下一轮交换 3 和 4...
```

### 3.2 代码

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
    ListNode* swapPairs(ListNode* head) {
        // 迭代法
        // 创造虚拟节点
        ListNode* dummy = new ListNode(0, head);
        // 创建 cur 节点（它后面两个节点需要交换，后面小于两个节点则停止）
        ListNode* cur = dummy;

        while (cur->next != nullptr && cur->next->next != nullptr) {
            // 定义待翻转节点 node1、node2
            ListNode* node1 = cur->next;
            ListNode* node2 = cur->next->next;

            // 执行翻转逻辑
            cur->next = node2;
            node1->next = node2->next;
            node2->next = node1;

            // cur + 2，移动到下一对的前一个
            cur = cur->next->next;
            // 或者 cur = node1;
        }

        return dummy->next;
    }
};
```

### 3.3 执行流程可视化

```
输入: [1, 2, 3, 4]

初始: dummy → 1 → 2 → 3 → 4
      ↑
     prev

第1轮交换 (1, 2):
  first = 1, second = 2

  交换前: dummy → 1 → 2 → 3 → 4

  prev->next = 2       → dummy → 2
  first->next = 3      → 1 → 3
  second->next = 1     → 2 → 1

  交换后: dummy → 2 → 1 → 3 → 4
                    ↑
                   prev

第2轮交换 (3, 4):
  first = 3, second = 4

  交换前: dummy → 2 → 1 → 3 → 4

  prev->next = 4       → 1 → 4
  first->next = null   → 3 → null
  second->next = 3     → 4 → 3

  交换后: dummy → 2 → 1 → 4 → 3 → null
                              ↑
                             prev

prev->next = null，循环结束

返回: dummy->next = [2, 1, 4, 3]
```

---

## 四、方法二：递归法 ⭐ 简洁优雅

### 4.1 思路

递归三部曲：
1. **终止条件**：链表为空或只有一个节点，无需交换
2. **返回值**：交换完成后的子链表头节点
3. **本级任务**：交换前两个节点，递归处理后面的链表

```
原链表: 1 → 2 → 3 → 4 → 5 → 6

分解:
  交换 (1, 2)，后面的 [3, 4, 5, 6] 递归处理

  递归返回: [3, 4, 5, 6] 变成 [4, 3, 6, 5]

  连接: 2 → 1 → [4, 3, 6, 5]

  结果: [2, 1, 4, 3, 6, 5]
```

### 4.2 代码

```cpp
class Solution {
public:
    ListNode* swapPairs(ListNode* head) {
        // 终止条件：空链表或只有一个节点
        if (head == nullptr || head->next == nullptr) {
            return head;
        }

        // 保存第二个节点（交换后会成为新的头）
        ListNode* second = head->next;

        // 递归处理后面的链表（从第3个节点开始）
        head->next = swapPairs(second->next);

        // 第二个节点指向第一个（完成交换）
        second->next = head;

        // 返回新的头节点（原来的第二个）
        return second;
    }
};
```

### 4.3 执行流程可视化

```
输入: [1, 2, 3, 4]

swapPairs(1):
  second = 2
  head->next = swapPairs(3)  ← 递归

    swapPairs(3):
      second = 4
      head->next = swapPairs(5)  ← 递归

        swapPairs(5):
          5->next = null，只有一个节点
          return 5

      4->next = 5
      return 4

  1->next = 4      (1 指向递归返回的结果 [4, 5])
  2->next = 1      (2 指向 1)
  return 2         (返回 [2, 1, 4, 5])

结果: [2, 1, 4, 5]
```

---

## 五、两种方法对比

```
┌─────────────────────────────────────────────────────────────────────┐
│  迭代法（哑节点）                     递归法                          │
│                                                                     │
│  时间: O(n)                           时间: O(n)                     │
│  空间: O(1)                           空间: O(n)（递归栈）            │
│                                                                     │
│  代码: 稍长                           代码: 极简（4行核心）            │
│  理解: 直观                           理解: 需递归思维               │
│  面试: 推荐 ⭐⭐⭐                      面试: 展示能力 ⭐⭐⭐⭐          │
│                                                                     │
│  适用: 生产环境                       适用: 面试展示                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 六、复杂度分析

| 方法 | 时间 | 空间 | 说明 |
|------|------|------|------|
| 迭代法 | **O(n)** | **O(1)** | 最优空间 |
| 递归法 | **O(n)** | **O(n)** | 递归栈深度 |

---

## 七、易错点 & 注意

| 坑点 | 说明 |
|------|------|
| **指针修改顺序** | 先保存 `second->next`，否则断链后找不到后面的节点 |
| **循环条件** | `prev->next && prev->next->next`，确保后面有两个节点 |
| **prev 移动** | 交换后 `prev = first`（不是 `prev = second`） |
| **递归终止** | `head == nullptr \|\| head->next == nullptr`，两个都要判断 |
| **返回值** | 递归返回的是**新的头节点**（second），不是原来的 head |

---

## 八、记忆口诀

> **"两两交换链表节点，迭代递归都能行；哑节点做前驱，三指针来交换；递归先处理后面，再连前面更简洁"**

---

## 九、面试话术模板

> "这道题我提供两种解法。迭代法使用哑节点，用 prev 指向待交换节点对的前一个，循环交换并移动指针，时间 O(n) 空间 O(1)。递归法更简洁：终止条件是空或单节点，保存第二个节点，递归处理后面链表，然后让第二个指向第一个，返回第二个作为新头。时间 O(n) 空间 O(n)。"

---

*整理日期：2026-07-29*
