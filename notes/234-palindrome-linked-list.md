# LeetCode 234 - 回文链表（Palindrome Linked List）

> **标签**：链表、双指针、递归、栈  
> **难度**：简单  
> **核心考点**：链表反转、快慢指针找中点

---

## 一、题目描述

给你一个单链表的头节点 `head`，请判断该链表是否为**回文链表**。如果是，返回 `true`；否则，返回 `false`。

**示例**：
```
输入: head = [1, 2, 2, 1]
输出: true

输入: head = [1, 2]
输出: false
```

---

## 二、核心思路

回文的特点是**正读反读都相同**。链表无法像数组那样随机访问，所以需要特殊处理。

### 2.1 常见方法

| 方法 | 时间 | 空间 | 说明 |
|------|------|------|------|
| 数组法 | O(n) | O(n) | 拷贝到数组，双指针比较 |
| 栈法 | O(n) | O(n) | 入栈后出栈比较 |
| 递归法 | O(n) | O(n) | 递归到末尾再比较 |
| 反转后半段 ⭐ | O(n) | O(1) | 最优，面试推荐 |

---

## 三、代码实现

### 3.1 数组法（你的代码）

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
    bool isPalindrome(ListNode* head) {
        vector<int> v;
        ListNode* cur = head;

        // 遍历链表，将值存入数组
        while (cur != nullptr) {
            v.push_back(cur->val);
            cur = cur->next;
        }

        // 双指针从两端向中间比较
        int left = 0;
        int right = v.size() - 1;

        while (left <= right) {
            if (v[left] != v[right]) {
                return false;  // 不对称，不是回文
            }
            left++;
            right--;
        }

        return true;  // 全部对称，是回文
    }
};
```

### 3.2 反转后半段法（O(1) 空间）⭐

```cpp
class Solution {
public:
    bool isPalindrome(ListNode* head) {
        if (head == nullptr || head->next == nullptr) return true;

        // 步骤1: 快慢指针找中点
        ListNode* slow = head;
        ListNode* fast = head;
        while (fast != nullptr && fast->next != nullptr) {
            slow = slow->next;      // 慢指针走1步
            fast = fast->next->next; // 快指针走2步
        }
        // slow 指向中点（奇数时偏右）

        // 步骤2: 反转后半段
        ListNode* secondHalf = reverseList(slow);

        // 步骤3: 比较前半段和反转后的后半段
        ListNode* firstHalf = head;
        ListNode* p = secondHalf;
        bool result = true;

        while (p != nullptr) {
            if (firstHalf->val != p->val) {
                result = false;
                break;
            }
            firstHalf = firstHalf->next;
            p = p->next;
        }

        // 步骤4: 恢复链表（可选）
        reverseList(secondHalf);

        return result;
    }

private:
    // 反转链表
    ListNode* reverseList(ListNode* head) {
        ListNode* prev = nullptr;
        ListNode* cur = head;
        while (cur != nullptr) {
            ListNode* next = cur->next;
            cur->next = prev;
            prev = cur;
            cur = next;
        }
        return prev;
    }
};
```

---

## 四、执行流程可视化

### 数组法

```
链表: 1 → 2 → 2 → 1

步骤1: 存入数组
  v = [1, 2, 2, 1]

步骤2: 双指针比较
  left=0, right=3: v[0]=1, v[3]=1, 相等 ✓
  left=1, right=2: v[1]=2, v[2]=2, 相等 ✓
  left=2, right=1: left > right, 结束

返回: true
```

### 反转后半段法

```
链表: 1 → 2 → 3 → 2 → 1

步骤1: 快慢指针找中点
  slow: 1 → 2 → 3
  fast: 1 → 3 → 1 → null
  slow 指向 3（中点）

步骤2: 反转后半段
  原: 3 → 2 → 1
  反: 1 → 2 → 3

步骤3: 比较
  前半: 1 → 2
  后半: 1 → 2
  1==1 ✓, 2==2 ✓

返回: true

步骤4: 恢复链表（再反转一次）
```

---

## 五、复杂度分析

| 方法 | 时间 | 空间 | 说明 |
|------|------|------|------|
| 数组法 | **O(n)** | **O(n)** | 需要额外数组 |
| 栈法 | **O(n)** | **O(n)** | 需要栈空间 |
| 递归法 | **O(n)** | **O(n)** | 递归栈空间 |
| 反转后半段 | **O(n)** | **O(1)** | ⭐ 最优 |

---

## 六、易错点 & 注意

| 坑点 | 说明 |
|------|------|
| **快慢指针终止条件** | `fast != nullptr && fast->next != nullptr`，注意是两个条件 |
| **奇数长度链表** | 中点节点属于后半段，反转后比较时前半段会少一个节点，不影响结果 |
| **恢复链表** | 面试时如果要求不能修改链表，需要再反转一次恢复 |
| **空链表/单节点** | 都是回文，直接返回 `true` |

---

## 七、举一反三

| 题号 | 题目 | 关联点 |
|------|------|--------|
| 206 | 反转链表 | 反转后半段的基础 |
| 876 | 链表的中间结点 | 快慢指针找中点 |
| 143 | 重排链表 | 找中点 + 反转 + 合并 |
| 125 | 验证回文串 | 数组/字符串版的回文判断 |

---

## 八、记忆口诀

> **"回文链表判对称，数组拷贝双指针；最优反转后半段，快慢中点再反比"**

---

## 九、面试话术模板

> "这道题我提供两种解法：数组法和 O(1) 空间法。数组法遍历链表存入数组，然后双指针从两端向中间比较，时间 O(n)，空间 O(n)。最优解是用快慢指针找到中点，反转后半段链表，然后比较前半段和反转后的后半段，最后再把链表恢复，时间 O(n)，空间 O(1)。"

---

*整理日期：2026-07-28*
