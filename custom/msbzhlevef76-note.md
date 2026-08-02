# 穿针引线法：

==先创建假头节点==`ListNode *dummy = new ListNode(-1);`

==放在真头节点前面==`dummy->next = head;`

- ==这样做的意义是不用单独处理 `left == 1` 的情况了==

创建一个before节点，指向要反转的第一个节点前一个节点，固定不动

- before先指向dummy,再循环后移left-1步

创建一个start节点指向要反转的第一个节点，也全程固定不动

循环（right - left）次

- 创建一个要移动的节点moveNode，先指向start的下一个
- start跳过moveNode，指向moveNode的下一个节点
- 把moveNode接在before后面一个节点的前面
- before连上moveNode

返回真头dummy->next

---



## 再画一遍图（用新命名）

```plain
初始：dummy → 1 → 2 → 3 → 4 → 5
             ↑   ↑
           before start
```

把 `3` 提到前面：

```cpp
moveNode = start->next;        // moveNode = 3

start->next = moveNode->next;  // 2 → 4（2 甩开 3）
moveNode->next = before->next; // 3 → 2（3 插到 1 后面，即 start 前面）
before->next = moveNode;       // 1 → 3（1 抱住 3）
```

```plain
结果：dummy → 1 → 3 → 2 → 4 → 5
             ↑       ↑
           before  start
```

------

## 核心动作一句话

> **每次把 `start` 后面的那个节点（`moveNode`）拔出来，插到 `before` 的屁股后面。**

`start` 始终指向原来的 `left` 位置（节点2），它就像个**桩子**，后面的节点逐个往前插队，最后 `start` 自然就成了反转后的尾节点。

---

# 完整代码

```c++
class Solution {
public:
    ListNode *reverseBetween(ListNode *head, int left, int right) {
        ListNode *dummy = new ListNode(-1);
        dummy->next = head;
        
        // 1. before 走到 left 的前一个节点，固定不动
        ListNode *before = dummy;
        for (int i = 0; i < left - 1; i++) {
            before = before->next;
        }
        
        // 2. start 是反转段的第一个节点，全程固定不动（反转后的尾巴）
        ListNode *start = before->next;
        
        // 3. 把 start 后面的节点逐个提到 before 后面，执行 right-left 次
        for (int i = 0; i < right - left; i++) {
            ListNode *moveNode = start->next;    // 要移动的节点
            
            start->next = moveNode->next;        // start 跳过 moveNode
            moveNode->next = before->next;       // moveNode 插到 before 后面
            before->next = moveNode;             // before 连上 moveNode
        }
        
        return dummy->next;
    }
};
```

