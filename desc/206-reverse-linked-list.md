```
class Solution {
public:
    ListNode* reverseList(ListNode* head) {
        ListNode* prev=nullptr;
        ListNode* cur=head;

        while(cur!=nullptr)
        {
            ListNode* next = cur->next;
            cur->next=prev;
            prev=cur;
            cur=next;
        }
        return prev;
    }
};
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