# LeetCode 138 - 复制带随机指针的链表（Copy List with Random Pointer）

> **标签**：哈希表、链表  
> **难度**：中等  
> **核心考点**：深拷贝、哈希表映射、链表复制

---

## 一、题目描述

给你一个长度为 `n` 的链表，每个节点包含一个额外增加的随机指针 `random`，该指针可以指向链表中的任何节点或空节点。

构造这个链表的**深拷贝**。深拷贝应该正好由 `n` 个**全新**节点组成，其中每个新节点的值都设为其对应的原节点的值。新节点的 `next` 指针和 `random` 指针也都应指向复制链表中的新节点，并使原链表和复制链表中的这些指针能够表示相同的链表状态。**复制链表中的指针都不应指向原链表中的节点**。

**示例**：
```
输入: head = [[7,null],[13,0],[11,4],[10,2],[1,0]]
输出: [[7,null],[13,0],[11,4],[10,2],[1,0]]

解释: 原链表有5个节点，random 指针指向任意位置
     深拷贝后，新链表的节点值和指针关系与原链表完全一致
     但新链表的节点是全新创建的，与原链表无关
```

---

## 二、为什么这道题比普通链表复制难？

### 2.1 普通链表复制

```
原链表: 1 → 2 → 3 → 4

复制:   1' → 2' → 3' → 4'

只需要按顺序创建新节点，连接 next 即可。
```

### 2.2 带 random 指针的链表复制

```
原链表: 
  7 → 13 → 11
  ↓   ↓    ↓
 null 7    1

random 指向的节点可能在前面、后面，甚至为空！

问题：
1. 复制 13 时，13.random 指向 7，但此时 7 的新节点还没创建？
   不对，从头开始复制，7 的新节点已经创建了。

2. 复制 7 时，7.random 指向 null，没问题。

3. 复制 11 时，11.random 指向 1，但 1 还没复制！
   此时不知道 1 的新节点地址是什么！
```

**核心难点**：`random` 指针可以指向**任意位置**（包括还没复制的节点），无法在单次遍历中确定所有 `random` 指针。

### 2.3 解决方案

| 方法 | 思路 | 时间 | 空间 |
|------|------|------|------|
| **哈希表法** ⭐ | 先创建所有节点，再统一连接 | O(n) | O(n) |
| **拼接法** | 原节点后面插入新节点，再拆分 | O(n) | O(1) |

---

## 三、方法一：哈希表法（你的代码）⭐

### 3.1 思路

**两遍遍历**：
1. **第一遍**：遍历原链表，创建新节点（只设 `val`），建立**原节点 → 新节点**的映射
2. **第二遍**：再次遍历，根据映射关系设置 `next` 和 `random`

```
原链表: 7 → 13 → 11
        ↓    ↓    ↓
       null  7    1

第一遍：创建映射
  map[7] = 7'   (新节点，val=7)
  map[13] = 13' (新节点，val=13)
  map[11] = 11' (新节点，val=11)

第二遍：建立连接
  map[7]->next = map[13]   → 7' → 13'
  map[7]->random = map[null] → null

  map[13]->next = map[11]  → 13' → 11'
  map[13]->random = map[7] → 13'.random = 7'

  map[11]->next = map[null] → null
  map[11]->random = map[1] → 但 1 不在 map 中！

  等等，原链表只有 3 个节点，1 是 random 指向的值？
  不，random 指向的是节点，不是值。
```

### 3.2 代码

```cpp
/*
// Definition for a Node.
class Node {
public:
    int val;
    Node* next;
    Node* random;

    Node(int _val) {
        val = _val;
        next = NULL;
        random = NULL;
    }
};
*/

class Solution {
public:
    Node* copyRandomList(Node* head) {
        // 哈希表法
        // 创建一个 key 和 value 都是节点指针的哈希表
        unordered_map<Node*, Node*> map;

        Node* cur = head;

        // 遍历原链表，创建一个没有连接关系只有 val 对应的新链表节点
        // 与源节点一一对应
        while (cur != nullptr) {
            map[cur] = new Node(cur->val);
            cur = cur->next;
        }

        // 建立新链表节点的链接关系
        cur = head;
        while (cur != nullptr) {   
            // 原链表节点对应的新节点的 next：
            // 链接原节点下一个节点对应的新节点
            // （既根据原链表找到了连接关系，又是基于新节点建立的链接）
            map[cur]->next = map[cur->next];

            // random 同理
            map[cur]->random = map[cur->random];

            cur = cur->next;
        }

        // 返回新链表头结点
        return map[head];
    }
};
```

### 3.3 执行流程可视化

```
原链表:
  节点A: val=7,  next=B, random=null
  节点B: val=13, next=C, random=A
  节点C: val=11, next=null, random=B

第一遍：创建映射
  map[A] = A' (val=7)
  map[B] = B' (val=13)
  map[C] = C' (val=11)

第二遍：建立连接
  cur=A:
    A'->next = map[B] = B'     → A' → B'
    A'->random = map[null] = null

  cur=B:
    B'->next = map[C] = C'     → B' → C'
    B'->random = map[A] = A'    → B'.random = A'

  cur=C:
    C'->next = map[null] = null
    C'->random = map[B] = B'    → C'.random = B'

结果: A' → B' → C'
      ↓     ↓     ↓
     null  A'    B'

返回: map[A] = A'
```

---

## 四、方法二：拼接法（O(1) 空间）

### 4.1 思路

不借助哈希表，在原链表每个节点后面插入新节点，最后拆分。

```
步骤1: 在每个原节点后面插入新节点
  A → A' → B → B' → C → C'

步骤2: 设置 random 指针
  A'.random = A.random.next  (A.random 是 X，X.next 就是 X')

步骤3: 拆分链表
  原链表: A → B → C
  新链表: A' → B' → C'
```

### 4.2 代码

```cpp
class Solution {
public:
    Node* copyRandomList(Node* head) {
        if (head == nullptr) return nullptr;

        // 步骤1: 在每个原节点后面插入新节点
        Node* cur = head;
        while (cur != nullptr) {
            Node* newNode = new Node(cur->val);
            newNode->next = cur->next;
            cur->next = newNode;
            cur = newNode->next;
        }

        // 步骤2: 设置 random 指针
        cur = head;
        while (cur != nullptr) {
            if (cur->random != nullptr) {
                cur->next->random = cur->random->next;
            }
            cur = cur->next->next;
        }

        // 步骤3: 拆分链表
        Node* newHead = head->next;
        cur = head;
        while (cur != nullptr) {
            Node* copy = cur->next;
            cur->next = copy->next;
            if (copy->next != nullptr) {
                copy->next = copy->next->next;
            }
            cur = cur->next;
        }

        return newHead;
    }
};
```

---

## 五、两种方法对比

```
┌─────────────────────────────────────────────────────────────────────┐
│  哈希表法                              拼接法                        │
│                                                                     │
│  时间: O(n)                            时间: O(n)                   │
│  空间: O(n)                            空间: O(1) ✅                 │
│                                                                     │
│  代码: 简洁直观                        代码: 稍复杂                  │
│  思路: 两遍遍历                        思路: 原地插入再拆分          │
│  面试: 推荐 ⭐⭐⭐                       面试: 进阶展示 ⭐⭐⭐⭐         │
│                                                                     │
│  核心: map[原节点] = 新节点             核心: 原节点->新节点->原节点   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 六、为什么这道题比普通链表复制难？

| 对比 | 普通链表复制 | 带 random 的链表复制 |
|------|-------------|---------------------|
| 指针类型 | 只有 `next` | `next` + `random` |
| 遍历顺序 | 顺序遍历即可 | `random` 可能指向未遍历的节点 |
| 关键问题 | 无 | 复制节点 A 时，A.random 指向的 B 可能还没创建 |
| 解决方案 | 顺序创建 | **哈希表预创建所有节点** 或 **原地插入** |

> **核心区别**：`random` 指针打破了链表的顺序性，使得"先创建节点再连接"的策略失效，必须**先知道所有节点的对应关系**，才能正确设置 `random`。

---

## 七、易错点 & 注意

| 坑点 | 说明 |
|------|------|
| **map[cur->next]** | 当 `cur->next` 为 `null` 时，`map[null]` 会创建新节点，需要处理 |
| **空链表** | `head == nullptr` 时直接返回 `nullptr` |
| **random 为 null** | `map[null]` 在 C++ 中会插入一个新键值对，注意处理 |
| **拼接法的拆分** | 注意恢复原链表，虽然题目不要求，但最好做 |

---

## 八、举一反三

| 题号 | 题目 | 关联点 |
|------|------|--------|
| 133 | 克隆图 | 图的深拷贝，同样用哈希表映射 |
| 148 | 排序链表 | 链表操作 |
| 206 | 反转链表 | 链表指针操作 |

---

## 九、记忆口诀

> **"复制链表带随机，哈希映射最清晰；先建节点再连针，原新对应不迷路；拼接法来 O(1) 空，原地插入再拆分"**

---

## 十、面试话术模板

> "这道题的关键在于 random 指针可以指向任意节点，包括还没复制的节点。我使用哈希表法：第一遍遍历创建所有新节点并建立原节点到新节点的映射；第二遍遍历根据原链表的 next 和 random 关系设置新链表的指针。时间 O(n)，空间 O(n)。如果要 O(1) 空间，可以用拼接法：在每个原节点后插入新节点，设置 random 后再拆分。"

---

*整理日期：2026-07-29*
