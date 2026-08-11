# 哈希集合法
```
class Solution {
public:
    bool hasCycle(ListNode *head) {
        unordered_set<ListNode*> set;
        ListNode* cur=head;
        while(cur!=nullptr)
        {
            if(set.count(cur))
            {
                return true;
            }
            set.insert(cur);
            cur=cur->next;
        }
        return false;
    }
};
```
时间：O(n)
空间：O(n)
---

# 快慢指针法
```
class Solution {
public:
    bool hasCycle(ListNode *head) {
        if(head==nullptr || head->next==nullptr)
        {
            return false;
        }
        ListNode* fast=head->next;
        ListNode* slow=head;

        while(slow!=fast)
        {
            if(fast==nullptr || fast->next==nullptr)
            {
                return false;
            }
            slow=slow->next;
            fast=fast->next->next;
        }
        return true;
    }
};
```
### 防止空指针解引用（Null Pointer Dereference）
第一处 `if(head==nullptr || head->next==nullptr)`

保护初始化：你的代码中` fast` 初始化为` head->next`。如果 head 本身是空指针，直接访问` head->next `就会导致程序崩溃。
提前返回：如果链表为空（0个节点）或只有一个节点，显然不可能形成环，直接返回 false 是最高效的做法。
第二处` if(fast==nullptr || fast->next==nullptr)`

保护快指针移动：在循环中，快指针每次要走两步，即执行` fast = fast->next->next`。
在执行这一步之前，必须确保` fast` 当前不为空，且它的下一个节点 `fast->next `也不为空。
**本质：防止出现`nullptr->next`**

时间：O(n)
空间：O(1)
---