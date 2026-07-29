# LeetCode 21 - 合并两个有序链表（Merge Two Sorted Lists）

> **标签**：链表、递归、双指针  
> **难度**：简单  
> **核心考点**：双指针合并、链表操作

---

## 一、题目描述

将两个**升序**链表合并为一个新的 **升序** 链表并返回。新链表是通过拼接给定的两个链表的所有节点组成的。

**示例**：
```
输入: list1 = [1, 2, 4], list2 = [1, 3, 4]
输出: [1, 1, 2, 3, 4, 4]

输入: list1 = [], list2 = []
输出: []

输入: list1 = [], list2 = [0]
输出: [0]
```

---

## 二、两种解法对比

| 解法 | 核心思想 | 时间复杂度 | 空间复杂度 | 推荐指数 |
|------|---------|-----------|-----------|---------|
| **方法一** | 双指针 + 新建节点 | O(n+m) | **O(n+m)** | ⭐⭐ 基础版 |
| **方法二** | 双指针 + 拼接原节点 | O(n+m) | **O(1)** | ⭐⭐⭐ 最优 |
| **方法三** | 递归 | O(n+m) | O(n+m) | ⭐⭐ 简洁 |

---

## 三、方法一：双指针 + 新建节点（你的代码）

### 3.1 思路

创建新链表，每次比较两个链表头，取较小值创建新节点，拼接到结果链表。

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
    ListNode* mergeTwoLists(ListNode* list1, ListNode* list2) {
        ListNode* dummy = new ListNode(0);
        ListNode* cur = dummy;

        while (list1 && list2) {
            if (list1->val < list2->val) {
                cur->next = new ListNode(list1->val);  // 新建节点
                list1 = list1->next;
                cur = cur->next;
            } else {
                cur->next = new ListNode(list2->val);  // 新建节点
                list2 = list2->next;
                cur = cur->next;
            }
        }

        if (list1 == nullptr) cur->next = list2;
        if (list2 == nullptr) cur->next = list1;

        return dummy->next;
    }
};
```

### 3.3 问题分析

**缺点**：每次都用 `new ListNode(...)` 创建新节点，**没有复用原链表的节点**。

- 空间复杂度：**O(n+m)**（新建了所有节点）
- 原链表的节点内存没有释放（LeetCode 不强制，但实际不好）

---

## 四、方法二：双指针 + 拼接原节点（最优）⭐

### 4.1 思路

不新建节点，直接**改变原链表节点的 next 指针**，把两个链表拼接起来。

### 4.2 代码

```cpp
class Solution {
public:
    ListNode* mergeTwoLists(ListNode* list1, ListNode* list2) {
        ListNode* dummy = new ListNode(0);
        ListNode* cur = dummy;

        while (list1 && list2) {
            if (list1->val < list2->val) {
                cur->next = list1;      // 直接拼接原节点
                list1 = list1->next;
            } else {
                cur->next = list2;      // 直接拼接原节点
                list2 = list2->next;
            }
            cur = cur->next;
        }

        // 连接剩余部分
        cur->next = list1 ? list1 : list2;

        return dummy->next;
    }
};
```

### 4.3 执行流程可视化

```
list1: 1 → 2 → 4
list2: 1 → 3 → 4

初始: dummy → nullptr
       ↑
      cur

第1轮: 1(list1) == 1(list2)，取 list2
  dummy → 1(list2)
           ↑
          cur
  list2: 3 → 4

第2轮: 1(list1) < 3(list2)，取 list1
  dummy → 1(list2) → 1(list1)
                      ↑
                     cur
  list1: 2 → 4

第3轮: 2(list1) < 3(list2)，取 list1
  dummy → 1 → 1 → 2
                  ↑
                 cur
  list1: 4

第4轮: 4(list1) > 3(list2)，取 list2
  dummy → 1 → 1 → 2 → 3
                      ↑
                     cur
  list2: 4

第5轮: 4(list1) == 4(list2)，取 list2
  dummy → 1 → 1 → 2 → 3 → 4(list2)
                              ↑
                             cur
  list2: null

list2 为空，cur->next = list1 (4)

结果: 1 → 1 → 2 → 3 → 4 → 4
```

---

## 五、方法三：递归法

### 5.1 思路

递归比较两个链表头，较小节点的 next 指向递归合并的结果。

### 5.2 代码

```cpp
class Solution {
public:
    ListNode* mergeTwoLists(ListNode* list1, ListNode* list2) {
        // 终止条件
        if (list1 == nullptr) return list2;
        if (list2 == nullptr) return list1;

        if (list1->val < list2->val) {
            list1->next = mergeTwoLists(list1->next, list2);
            return list1;
        } else {
            list2->next = mergeTwoLists(list1, list2->next);
            return list2;
        }
    }
};
```

---

## 六、三种方法对比总结

```
┌─────────────────────────────────────────────────────────────────────┐
│  新建节点法              拼接原节点法              递归法             │
│                                                                     │
│  cur->next = new Node(val)    cur->next = list1          递归比较   │
│                                                                     │
│  时间: O(n+m)                 时间: O(n+m)              时间: O(n+m)│
│  空间: O(n+m) ❌              空间: O(1) ✅             空间: O(n+m)│
│                                                                     │
│  新建所有节点                  复用原节点                  代码简洁   │
│  不推荐 ❌                    推荐 ⭐⭐⭐                 展示 ⭐⭐    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 七、复杂度分析

| 方法 | 时间复杂度 | 空间复杂度 | 说明 |
|------|-----------|-----------|------|
| **新建节点法** | **O(n+m)** | **O(n+m)** | 新建了所有节点 |
| **拼接原节点法** | **O(n+m)** | **O(1)** | ⭐ 只修改指针，最优 |
| **递归法** | **O(n+m)** | **O(n+m)** | 递归栈空间 |

> n 和 m 分别是两个链表的长度。

---

## 八、易错点 & 注意

| 坑点 | 说明 |
|------|------|
| **拼接 vs 新建** | 优先用拼接法，空间更优 |
| **循环条件** | `while (list1 && list2)`，不是 `\|\|` |
| **连接剩余** | 循环结束后，一个链表可能还有剩余，直接拼接 |
| **递归终止** | `if (list1 == nullptr) return list2`，处理空链表 |
| **返回值** | 返回 `dummy->next`，不是 `dummy` |

---

## 九、举一反三

| 题号 | 题目 | 关联点 |
|------|------|--------|
| 23 | 合并 K 个升序链表 | 分治合并，复用本题方法 |
| 148 | 排序链表 | 归并排序中的 merge 函数 |
| 88 | 合并两个有序数组 | 数组版的合并 |

---

## 十、记忆口诀

> **"合并有序链表，双指针来帮忙；拼接原节点最优，新建节点费空间；递归简洁栈空间，面试推荐拼接法"**

---

## 十一、面试话术模板

> "这道题我使用双指针解决。创建虚拟头节点，比较两个链表的头节点，把较小的节点拼接到结果链表。循环结束后把剩余部分直接接上。注意直接拼接原节点而不是新建节点，空间复杂度 O(1)。也可以用递归实现，代码更简洁但空间 O(n+m)。"

---

*整理日期：2026-07-29*
