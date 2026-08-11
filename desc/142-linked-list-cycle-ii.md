```
class Solution {
public:
    ListNode *detectCycle(ListNode *head) {
        unordered_set<ListNode*> set;
        ListNode* cur=head;
        while(cur!=nullptr)
        {
            if(set.count(cur))
            {
                return cur;
            }
            set.insert(cur);
            cur=cur->next;
        }
        return nullptr;
    }
};
```