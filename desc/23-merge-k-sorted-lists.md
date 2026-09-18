```cpp
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
public:
    // 自定义比较器：小根堆，val小的优先出队
    struct cmp {
        bool operator()(ListNode* a, ListNode* b) {
            return a->val > b->val; 
        }
    };

    ListNode* mergeKLists(vector<ListNode*>& lists) {
        // 优先队列：存链表节点指针
        priority_queue<ListNode*, vector<ListNode*>, cmp> heap;

        // 把所有非空链表的头节点入堆
        for (auto head : lists) {
            if (head != nullptr) {
                heap.push(head);
            }
        }

        // 虚拟头结点，简化链表拼接
        ListNode dummy;
        ListNode* cur = &dummy;

        while (!heap.empty()) {
            // 取出当前最小节点
            ListNode* minNode = heap.top();
            heap.pop();

            cur->next = minNode;
            cur = cur->next;

            // 如果这个节点后面还有节点，继续入堆
            if (minNode->next != nullptr) {
                heap.push(minNode->next);
            }
        }

        return dummy.next;
    }
};一个坑
普通内置类型：priority_queue<int,vector<int>,greater<int>> 直接小根堆 ✔
自定义结构体 / 指针：greater 不能直接拿来用，要自己定义比较规则 ✔
```

---
