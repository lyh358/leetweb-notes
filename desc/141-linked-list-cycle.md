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
