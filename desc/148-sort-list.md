# 
```
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode() : val(0), next(nullptr) {}
 *     ListNode(int x) : val(x), next(nullptr) {}
 *     ListNode(int x, ListNode *next) : val(x), next(next) {}
 * };
 */
class Solution {
private:
    //快慢指针找中间节点
    ListNode* findMiddleNode(ListNode* head)
    {
        ListNode* slow=head;
        ListNode* fast=head->next;

        while(fast!=nullptr && fast->next!=nullptr)
        {
            slow=slow->next;
            fast=fast->next->next;
        }
        return slow;
    }

    //两个链表排序后合并成一个链表
    ListNode* sortedMerge(ListNode* l1,ListNode* l2)
    {
        ListNode* dummy=new ListNode(0);
        ListNode* cur=dummy;

        while(l1 && l2)
        {
            if(l1->val < l2->val)
            {
                cur->next=l1;
                l1=l1->next;
            }
            else
            {
                cur->next=l2;
                l2=l2->next;
            }
            cur=cur->next;
        }
        if(l1) cur->next=l1;
        if(l2) cur ->next=l2;

        return dummy->next;
    }

public:
    ListNode* sortList(ListNode* head) {
        //边界检查
        if(head==nullptr || head->next==nullptr)
        {
            return head;
        }

        //找中间节点
        ListNode* mid = findMiddleNode(head);

        //分离右半链
        ListNode* righthead = mid->next;
        mid->next=nullptr;

        //递归排序左半链和右半链
        ListNode* left = sortList(head);
        ListNode* right = sortList(righthead);

        //合并两个左右有序列表
        return sortedMerge(left,right);
    }
};
```