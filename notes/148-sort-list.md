# LeetCode 148 - 排序链表（Sort List）

> **标签**：链表、排序、归并排序、分治  
> **难度**：中等  
> **核心考点**：链表归并排序、快慢指针找中点、合并有序链表

---

## 一、题目描述

给你链表的头结点 `head`，请将其按 **升序** 排列并返回 **排序后的链表**。

**要求**：
- 时间复杂度：**O(n log n)**
- 空间复杂度：尽可能 **O(1)**

**示例**：
```
输入: head = [4, 2, 1, 3]
输出: [1, 2, 3, 4]

输入: head = [-1, 5, 3, 4, 0]
输出: [-1, 0, 3, 4, 5]
```

---

## 二、两种解法对比

| 解法 | 时间 | 空间 | 面试接受度 | 说明 |
|------|------|------|-----------|------|
| **数组法** | O(n log n) | **O(n)** | ⭐⭐ | 简单但不符合链表特性 |
| **归并排序** ⭐ | O(n log n) | O(log n) 或 **O(1)** | ⭐⭐⭐⭐⭐ | 标准答案，面试推荐 |

---

## 三、方法一：数组法（基础解法）

### 3.1 思路

把链表值存入数组，排序后再赋值回去。

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
    ListNode* sortList(ListNode* head) {
        vector<int> v;
        ListNode* cur = head;

        // 遍历链表，存入数组
        while (cur != nullptr) {
            v.push_back(cur->val);
            cur = cur->next;
        }

        // 数组排序
        sort(v.begin(), v.end());

        // 排序后的值赋值回链表
        cur = head;
        int i = 0;
        while (cur != nullptr) {
            cur->val = v[i++];
            cur = cur->next;
        }

        return head;
    }
};
```

### 3.3 复杂度分析

| 指标 | 复杂度 | 原因 |
|------|--------|------|
| 时间 | **O(n log n)** | 遍历 O(n) + 排序 O(n log n) |
| 空间 | **O(n)** | 额外数组 |

### 3.4 优缺点

- ✅ 代码简单，容易实现
- ❌ 空间复杂度 O(n)，不符合链表原地操作特性
- ❌ 没有利用链表结构优势
- ⚠️ 面试时不推荐单独写这个，需要展示更优解

---

## 四、方法二：归并排序（面试标准答案）⭐

### 4.1 思路

链表归并排序 = **找中点** + **递归排序左右两半** + **合并两个有序链表**

```
[4, 2, 1, 3]

步骤1: 找中点
  快慢指针: slow=2, fast=null
  中点在 slow，断开为 [4, 2] 和 [1, 3]

步骤2: 递归排序
  sort([4, 2]) → 找中点 → [4] 和 [2] → 合并 → [2, 4]
  sort([1, 3]) → 找中点 → [1] 和 [3] → 合并 → [1, 3]

步骤3: 合并
  merge([2, 4], [1, 3]) → [1, 2, 3, 4]
```

### 4.2 代码

```cpp
class Solution {
public:
    ListNode* sortList(ListNode* head) {
        // 递归终止条件：空链表或单节点
        if (head == nullptr || head->next == nullptr) {
            return head;
        }

        // 步骤1: 快慢指针找中点
        ListNode* slow = head;
        ListNode* fast = head->next;  // fast 从 next 开始，slow 停在中间偏左

        while (fast != nullptr && fast->next != nullptr) {
            slow = slow->next;          // 慢指针走1步
            fast = fast->next->next;    // 快指针走2步
        }

        // 断开链表：左半 [head, slow]，右半 [mid, ...]
        ListNode* mid = slow->next;
        slow->next = nullptr;

        // 步骤2: 递归排序左右两半
        ListNode* left = sortList(head);
        ListNode* right = sortList(mid);

        // 步骤3: 合并两个有序链表
        return merge(left, right);
    }

private:
    // 合并两个有序链表（同 LeetCode 21）
    ListNode* merge(ListNode* l1, ListNode* l2) {
        ListNode dummy(0);      // 虚拟头节点
        ListNode* cur = &dummy;

        while (l1 != nullptr && l2 != nullptr) {
            if (l1->val < l2->val) {
                cur->next = l1;
                l1 = l1->next;
            } else {
                cur->next = l2;
                l2 = l2->next;
            }
            cur = cur->next;
        }

        // 连接剩余部分
        cur->next = (l1 != nullptr) ? l1 : l2;

        return dummy.next;
    }
};
```

### 4.3 执行流程可视化

```
输入: [4, 2, 1, 3]

sortList([4, 2, 1, 3]):
  找中点: slow=2, fast=null
  断开: [4, 2] 和 [1, 3]

  left = sortList([4, 2]):
    找中点: slow=4, fast=null
    断开: [4] 和 [2]

    left = sortList([4]) → [4]
    right = sortList([2]) → [2]
    merge([4], [2]) → [2, 4]

  right = sortList([1, 3]):
    找中点: slow=1, fast=null
    断开: [1] 和 [3]

    left = sortList([1]) → [1]
    right = sortList([3]) → [3]
    merge([1], [3]) → [1, 3]

  merge([2, 4], [1, 3]):
    2 vs 1 → 1
    2 vs 3 → 2
    4 vs 3 → 3
    4      → 4
    结果: [1, 2, 3, 4]
```

### 4.4 复杂度分析

| 指标 | 复杂度 | 原因 |
|------|--------|------|
| 时间 | **O(n log n)** | 每层合并 O(n)，共 log n 层 |
| 空间 | **O(log n)** | 递归栈深度（自顶向下） |

---

## 五、关键细节解析

### 5.1 快慢指针找中点

```cpp
ListNode* slow = head;
ListNode* fast = head->next;  // 关键！fast 从 next 开始

while (fast != nullptr && fast->next != nullptr) {
    slow = slow->next;
    fast = fast->next->next;
}
```

| fast 起始位置 | slow 最终位置 | 适用场景 |
|-------------|-------------|---------|
| `head` | 中间偏右（上中点） | 找中点 |
| `head->next` | 中间偏左（下中点） | **归并排序断开链表** |

> 归并排序需要把链表**均匀分成两半**，`fast` 从 `head->next` 开始能让 `slow` 停在**下中点**，左半部分 <= 右半部分。

### 5.2 断开链表

```cpp
ListNode* mid = slow->next;  // 右半部分的头
slow->next = nullptr;        // 断开！左半部分的尾指向 null
```

> 必须断开，否则递归时无法正确划分左右两半。

### 5.3 合并有序链表

同 LeetCode 21，使用**虚拟头节点**简化操作。

---

## 六、易错点 & 注意

| 坑点 | 说明 |
|------|------|
| **fast 起始位置** | 归并排序用 `fast = head->next`，不是 `head` |
| **断开链表** | `slow->next = nullptr` 不能忘，否则递归无限循环 |
| **递归终止条件** | `head == nullptr \|\| head->next == nullptr` |
| **合并时移动指针** | `cur->next = l1` 后，`l1 = l1->next`，不能忘 |

---

## 七、进阶：自底向上归并排序（O(1) 空间）

```cpp
class Solution {
public:
    ListNode* sortList(ListNode* head) {
        if (head == nullptr || head->next == nullptr) return head;

        // 计算链表长度
        int length = 0;
        ListNode* cur = head;
        while (cur != nullptr) {
            length++;
            cur = cur->next;
        }

        ListNode dummy(0, head);

        // 自底向上：step = 1, 2, 4, 8, ...
        for (int step = 1; step < length; step *= 2) {
            ListNode* prev = &dummy;
            cur = dummy.next;

            while (cur != nullptr) {
                // 切出左半部分 [cur, leftEnd]
                ListNode* left = cur;
                ListNode* right = split(left, step);
                cur = split(right, step);  // 下一轮的起点

                // 合并并接到 prev 后面
                prev = merge(left, right, prev);
            }
        }

        return dummy.next;
    }

private:
    // 从 head 开始切出 n 个节点，返回剩余部分的头
    ListNode* split(ListNode* head, int n) {
        for (int i = 1; i < n && head != nullptr; i++) {
            head = head->next;
        }
        if (head == nullptr) return nullptr;
        ListNode* rest = head->next;
        head->next = nullptr;
        return rest;
    }

    // 合并 l1 和 l2，接到 prev 后面，返回新的尾节点
    ListNode* merge(ListNode* l1, ListNode* l2, ListNode* prev) {
        ListNode* cur = prev;
        while (l1 != nullptr && l2 != nullptr) {
            if (l1->val < l2->val) {
                cur->next = l1;
                l1 = l1->next;
            } else {
                cur->next = l2;
                l2 = l2->next;
            }
            cur = cur->next;
        }
        cur->next = (l1 != nullptr) ? l1 : l2;
        while (cur->next != nullptr) cur = cur->next;
        return cur;
    }
};
```

> 自底向上不需要递归，空间 O(1)，但代码复杂，面试中写出自顶向下即可。

---

## 八、举一反三

| 题号 | 题目 | 关联点 |
|------|------|--------|
| 21 | 合并两个有序链表 | merge 函数 |
| 876 | 链表的中间结点 | 快慢指针找中点 |
| 143 | 重排链表 | 找中点 + 反转 + 合并 |
| 23 | 合并 K 个升序链表 | 分治合并 |

---

## 九、记忆口诀

> **"链表排序归并上，找中断开再递归；快慢指针分两半，合并有序双指针；自顶向下 O(log n)，自底向上 O(1) 空"**

---

## 十、面试话术模板

> "这道题我提供两种解法：基础解法是把值存入数组排序再赋值回去，时间 O(n log n)，空间 O(n)。更优的解法是**归并排序**：用快慢指针找中点，递归排序左右两半，然后合并两个有序链表。时间 O(n log n)，空间 O(log n)（递归栈）。如果要 O(1) 空间，可以用自底向上的迭代归并排序。"

---

*整理日期：2026-07-29*
