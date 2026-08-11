### 参考代码：哈希表法
用哈希集合存放去重的节点指针（唯一地址），遍历A链表初始化集合，然后遍历B链表，在集合中查找节点是否存在，第一个存在节点即为
```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode(int x) : val(x), next(NULL) {}
 * };
 */
class Solution {
public:
    ListNode *getIntersectionNode(ListNode *headA, ListNode *headB) {
        unordered_set<ListNode*> uset;
        ListNode *cur = headA;

        // 遍历链表 A，把所有节点存入哈希表
        while (cur != nullptr) {
            uset.insert(cur);
            cur = cur->next;
        }

        // 遍历链表 B，找第一个在哈希表中的节点
        cur = headB;
        while (cur != nullptr) {
            if (uset.count(cur)) {   // 为什么用 count 不用 find？
                return cur;
            }
            cur = cur->next;
        }

        return nullptr;
    }
};
```
