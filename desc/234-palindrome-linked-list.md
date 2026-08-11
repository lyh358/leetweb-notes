先cun

```
class Solution {
public:
    bool isPalindrome(ListNode* head) {
        vector<int> v;
        ListNode* cur=head;

        while(cur!=nullptr)
        {
            v.push_back(cur->val);
            cur=cur->next;
        }

        int left=0,right=v.size()-1;

        while(left<=right)
        {
            if(v[left]!=v[right])
            {
                return false;
            }
            left++;
            right--;
        }
        return true;
    }
};
```