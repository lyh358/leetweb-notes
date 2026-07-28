# C++ 链表知识点汇总

> 刷 LeetCode 链表专题前必备知识

---

## 一、链表基础概念

### 1.1 什么是链表

链表是一种**线性数据结构**，元素通过**指针**连接，不要求连续内存。

```
数组: [A][B][C][D]  ← 连续内存
      0  1  2  3

链表: [A]→[B]→[C]→[D]→∅  ← 非连续，指针连接
      head
```

### 1.2 链表 vs 数组

| 特性 | 数组 | 链表 |
|------|------|------|
| 内存 | 连续 | 不连续 |
| 访问 | O(1) 随机访问 | O(n) 顺序访问 |
| 插入/删除 | O(n) | O(1)（已知前驱） |
| 大小 | 固定 | 动态 |
| 缓存友好 | 好 | 差 |

---

## 二、C++ 链表定义

### 2.1 单链表节点定义（LeetCode 标准）

```cpp
// LeetCode 标准定义
struct ListNode {
    int val;           // 节点值
    ListNode *next;    // 指向下一个节点的指针

    // 构造函数
    ListNode() : val(0), next(nullptr) {}
    ListNode(int x) : val(x), next(nullptr) {}
    ListNode(int x, ListNode *next) : val(x), next(next) {}
};
```

### 2.2 创建节点

```cpp
// 方式1: 默认构造
ListNode* node1 = new ListNode();      // val=0, next=nullptr

// 方式2: 带值构造
ListNode* node2 = new ListNode(5);     // val=5, next=nullptr

// 方式3: 带值和next构造
ListNode* node3 = new ListNode(3, node2);  // val=3, next指向node2
```

### 2.3 连接节点

```cpp
ListNode* head = new ListNode(1);
head->next = new ListNode(2);
head->next->next = new ListNode(3);

// 链表: 1 → 2 → 3 → nullptr
```

### 2.4 遍历链表

```cpp
ListNode* cur = head;
while (cur != nullptr) {
    cout << cur->val << " ";
    cur = cur->next;  // 移动到下一个节点
}
// 输出: 1 2 3
```

> ⚠️ **注意**：遍历后 `cur` 指向 `nullptr`，`head` 还在原处。如果需要保留头节点，用临时指针遍历。

---

## 三、链表基本操作

### 3.1 插入节点

```cpp
// 在节点 prev 后面插入新节点（O(1)）
ListNode* prev = ...;           // 已知前驱节点
ListNode* newNode = new ListNode(10);

newNode->next = prev->next;     // 新节点指向原后继
prev->next = newNode;           // 前驱指向新节点

// 原: prev → nextNode
// 后: prev → newNode → nextNode
```

### 3.2 删除节点

```cpp
// 删除 prev 后面的节点（O(1)）
ListNode* prev = ...;
ListNode* toDelete = prev->next;

prev->next = toDelete->next;    // 跳过要删除的节点
// 或者: prev->next = prev->next->next;

delete toDelete;                // 释放内存（LeetCode一般不需要）
```

### 3.3 头插法（逆序构建链表）

```cpp
ListNode* head = nullptr;
for (int i = 1; i <= 5; i++) {
    ListNode* newNode = new ListNode(i);
    newNode->next = head;   // 新节点指向原头
    head = newNode;         // 更新头指针
}
// 结果: 5 → 4 → 3 → 2 → 1
```

### 3.4 尾插法（顺序构建链表）

```cpp
ListNode* dummy = new ListNode(0);  // 虚拟头节点
ListNode* tail = dummy;

for (int i = 1; i <= 5; i++) {
    tail->next = new ListNode(i);
    tail = tail->next;
}

ListNode* head = dummy->next;  // 真正的头节点
delete dummy;
// 结果: 1 → 2 → 3 → 4 → 5
```

---

## 四、链表核心技巧

### 4.1 虚拟头节点（Dummy Node）⭐⭐⭐

**作用**：统一头节点和其他节点的处理逻辑，避免特殊判断。

```cpp
// 删除链表中值为 val 的所有节点
ListNode* removeElements(ListNode* head, int val) {
    ListNode* dummy = new ListNode(0, head);  // 虚拟头指向真实头
    ListNode* cur = dummy;

    while (cur->next != nullptr) {
        if (cur->next->val == val) {
            ListNode* temp = cur->next;
            cur->next = cur->next->next;
            delete temp;
        } else {
            cur = cur->next;
        }
    }

    ListNode* newHead = dummy->next;
    delete dummy;
    return newHead;
}
```

> 没有虚拟头节点时，删除头节点需要特殊处理；有了虚拟头，所有节点一视同仁。

### 4.2 快慢指针（Floyd 判圈法）⭐⭐⭐

```cpp
// 判断链表是否有环
bool hasCycle(ListNode* head) {
    ListNode* slow = head;  // 慢指针：每次走1步
    ListNode* fast = head;  // 快指针：每次走2步

    while (fast != nullptr && fast->next != nullptr) {
        slow = slow->next;
        fast = fast->next->next;

        if (slow == fast) {  // 相遇，有环
            return true;
        }
    }
    return false;  // fast 到达末尾，无环
}
```

**快慢指针的应用**：
- 判断环（相遇则有环）
- 找环入口（相遇后，一个回到头，同步走，再次相遇即入口）
- 找中点（快指针到末尾时，慢指针在中点）
- 找倒数第 k 个节点（快指针先走 k 步）

### 4.3 链表反转 ⭐⭐⭐

```cpp
// 迭代法反转链表
ListNode* reverseList(ListNode* head) {
    ListNode* prev = nullptr;   // 前一个节点
    ListNode* cur = head;       // 当前节点

    while (cur != nullptr) {
        ListNode* next = cur->next;  // 暂存下一个
        cur->next = prev;             // 反转指向
        prev = cur;                   // 前移 prev
        cur = next;                   // 前移 cur
    }

    return prev;  // 新的头节点
}
```

**过程可视化**：
```
初始: 1 → 2 → 3 → 4 → nullptr
      ↑
     head

步骤1: nullptr ← 1   2 → 3 → 4
              prev  cur/next

步骤2: nullptr ← 1 ← 2   3 → 4
                   prev  cur/next

步骤3: nullptr ← 1 ← 2 ← 3   4
                        prev  cur/next

步骤4: nullptr ← 1 ← 2 ← 3 ← 4
                             prev  cur=nullptr

返回 prev = 4（新头节点）
```

### 4.4 递归法反转链表

```cpp
ListNode* reverseList(ListNode* head) {
    // 递归终止条件
    if (head == nullptr || head->next == nullptr) {
        return head;
    }

    // 反转后面的链表
    ListNode* newHead = reverseList(head->next);

    // 当前节点的下一个节点指向当前节点（反转）
    head->next->next = head;
    head->next = nullptr;  // 防止环

    return newHead;
}
```

### 4.5 双指针技巧

```cpp
// 找链表中点（快2步，慢1步）
ListNode* findMiddle(ListNode* head) {
    ListNode* slow = head;
    ListNode* fast = head;

    while (fast != nullptr && fast->next != nullptr) {
        slow = slow->next;
        fast = fast->next->next;
    }

    return slow;  // 中点（偶数时偏右）
}

// 找倒数第 k 个节点
ListNode* findKthFromEnd(ListNode* head, int k) {
    ListNode* fast = head;
    ListNode* slow = head;

    // fast 先走 k 步
    for (int i = 0; i < k; i++) {
        fast = fast->next;
    }

    // 同步走，fast 到末尾时 slow 在倒数第 k 个
    while (fast != nullptr) {
        fast = fast->next;
        slow = slow->next;
    }

    return slow;
}
```

---

## 五、链表常见题型

### 5.1 基础操作

| 题号 | 题目 | 考点 |
|------|------|------|
| 203 | 移除链表元素 | 虚拟头节点 |
| 206 | 反转链表 | 链表反转（迭代/递归） |
| 707 | 设计链表 | 链表基本操作 |

### 5.2 双指针

| 题号 | 题目 | 考点 |
|------|------|------|
| 19 | 删除链表的倒数第 N 个结点 | 快慢指针 |
| 876 | 链表的中间结点 | 快慢指针找中点 |
| 141 | 环形链表 | 快慢指针判环 |
| 142 | 环形链表 II | 快慢指针找环入口 |
| 160 | 相交链表 | 双指针找交点 |

### 5.3 反转相关

| 题号 | 题目 | 考点 |
|------|------|------|
| 92 | 反转链表 II | 反转指定区间 |
| 25 | K 个一组翻转链表 | 分组反转 |
| 234 | 回文链表 | 反转 + 比较 |

### 5.4 合并

| 题号 | 题目 | 考点 |
|------|------|------|
| 21 | 合并两个有序链表 | 双指针合并 |
| 23 | 合并 K 个升序链表 | 优先队列/分治 |

### 5.5 重排

| 题号 | 题目 | 考点 |
|------|------|------|
| 143 | 重排链表 | 找中点 + 反转 + 合并 |
| 148 | 排序链表 | 归并排序 |

---

## 六、链表题解题套路

### 6.1 通用步骤

```
1. 是否需要虚拟头节点？
   → 涉及头节点修改（删除、插入）时，建议用

2. 是否需要快慢指针？
   → 找中点、判环、找倒数第 k 个时，用

3. 是否需要反转？
   → 题目涉及逆序、回文、重排时，考虑反转

4. 注意边界条件
   → 空链表、单节点、头节点/尾节点处理
```

### 6.2 画图辅助

链表题一定要**画图**！在纸上画出指针的变化过程，避免混乱。

```
初始: 1 → 2 → 3 → 4 → 5
      ↑
     head

操作后标注每个指针的位置，确保逻辑正确。
```

---

## 七、易错点 & 注意

| 坑点 | 说明 |
|------|------|
| **空指针访问** | `cur->next` 前确保 `cur != nullptr` |
| **内存泄漏** | 删除节点后记得 `delete`（LeetCode 一般不管） |
| **断链** | 反转或删除时，先保存 `next` 再操作 |
| **返回头节点** | 操作后头节点可能改变，注意返回 |
| **环** | 反转时最后节点的 `next` 要置 `nullptr` |

---

## 八、记忆口诀

> **"链表题，画图先；虚拟头，少判断；快慢针，找中点；双指针，合并便；要反转，三指针；断链前，保存先"**

---

*整理日期：2026-07-28*
