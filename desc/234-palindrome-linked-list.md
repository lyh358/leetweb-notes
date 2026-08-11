**先存数组，再用普通的双指针逼近回文判断方法即可**

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

        while(left<=right)//奇数数组，可以相交
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