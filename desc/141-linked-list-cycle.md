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


时间：O(n)
空间：O(1)
---